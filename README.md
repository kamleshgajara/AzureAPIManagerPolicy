# AzureAPIManagerPolicy


How to use Azure API manager with SPA application authenticated by non-Azure identity providers (including onprem IDPs) using implicit grant.

We have an application that requires to use corporate authentication for its userbase, this application is hosted in Azure public cloud and IDP is On Prem. UI is simple react based application deployed on Azure blob storage and served through CDN. At the same time all APIs hosted inside AKS as different Microservices. As per our requirements users should be authenticated against the corporate authentication provider.

Simple architecture Diagram:
 

Sequence Diagram (Including OIDC)

 


Our immediate solution was to use Open ID connect protocol (OIDC) to have user authentication. OIDC is pretty straight forward that was not the problem, problem was for SPA (Single page application) is how to validate implicit grant token issued by identity provider for every subsequent calls to our microservices using API manager. After looking at all the documentation from Microsoft we noticed that all the documentation provided by Azure with OIDC or oAuth2 is based on Azure AD or Azure ADB2C. 

This is how we mitigated this problem.
•	Created custom policies in Azuer API manager 
•	Every request to get data from our Microservices, SPA needed to pass implicit grant token in header.
•	API manager looks for this header and checks in local cache to see we already validate the token with IDP.
•	If its new token and its not validated before 
o	API manager makes a REST calls to IDP on token validation /introspection endpoint (rfc7662)
o	If token is valid, IDP returns JWT
o	If tome in invalid or expired, it returns active=false.
•	API manager parse the response from IDP and checks if token is valid or not. 
•	If the token is not valid, API manager will return 401 to SPA.
•	If token is valid API manager stores token and its expiry in local cache and forward the calls to backend microservices.
•	Next time, when SPA needs data from other endpoint/microservice it returns same implicit grant token and this time API manager will have this token in cache it validates its expiry and it will forward the calls to backend APIs.

Complete Policy defined for our API manager is given below.


<policies>
    <inbound>
        <!-- Extract Token from Authorization header parameter -->
        <set-variable name="token" value="@(context.Request.Headers.GetValueOrDefault("Authorization","scheme param").Split(' ').Last())" />
        <!-- Check if the token variable is empty or null and return 401 Unauthorized if that is the case -->
        <choose>
            <when condition="@(System.String.IsNullOrEmpty((string)context.Variables["token"]))">
                <return-response response-variable-name="responseVariableName">
                    <set-status code="401" reason="Unauthorized" />
                    <set-header name="WWW-Authenticate" exists-action="override">
                        <value>Bearer, error="invalid_token"</value>
                    </set-header>
                </return-response>
            </when>
        </choose>
        <!-- Check if there is a previous value in the cache for this token -->
        <cache-lookup-value key="@((string)context.Variables["token"])" variable-name="introspectToken" />
        <choose>
            <!-- If we don’t find it in the cache, make a request for it and store it -->
            <when condition="@(!context.Variables.ContainsKey("introspectToken"))">
                <!--Send request to Token Server to validate token (see RFC 7662) -->
                <send-request mode="new" response-variable-name="tokenstate" timeout="20" ignore-error="true">
                    <set-url>https://yourIDPprovider.com/as/introspect.oauth2</set-url>
                    <set-method>POST</set-method>
                    <set-header name="Authorization" exists-action="override">
                        <value>Basic base64(ClientID:Secret)value>
                    </set-header>
                    <set-header name="Content-Type" exists-action="override">
                        <value>application/x-www-form-urlencoded</value>
                    </set-header>
                    <set-body>@($"token={(string)context.Variables["token"]}")</set-body>
                </send-request>
                <!-- cache the response of the Token Server with the AT as the key -->
                <cache-store-value key="@((string)context.Variables["token"])" value="@(((IResponse)context.Variables["tokenstate"]))" duration="@{
                    var header = ((IResponse)context.Variables["tokenstate"]).Headers.GetValueOrDefault("Cache-Control","");
                    var maxAge = Regex.Match(header, @"max-age=(?<maxAge>\d+)").Groups["maxAge"]?.Value;
                    return (!string.IsNullOrEmpty(maxAge))?int.Parse(maxAge):300;
                    }" />
            </when>
        </choose>
        <!-- Query the cache for a value with key AT and store the data in a context variable "introspectToken" -->
        <cache-lookup-value key="@((string)context.Variables["token"])" variable-name="introspectToken" />
        <choose>
            <!--Check active property in response -->
            <when condition="@((bool)((IResponse)context.Variables["introspectToken"]).Body.As<JObject>()["active"] == false)">
                <!-- active is equal to false, return 401 Unauthorized -->
                <return-response response-variable-name="responseVariableName">
                    <set-status code="401" reason="Unauthorized" />
                    <set-header name="WWW-Authenticate" exists-action="override">
                        <value>Bearer, error="invalid_token"</value>
                    </set-header>
                </return-response>
            </when>
            <otherwise>
                <!-- Response contains active=true replace AT with JWT-->
                <cache-lookup-value key="@((string)context.Variables["token"])" variable-name="introspectToken" />
                <set-header name="Authorization" exists-action="override">
                    <value>@($"Bearer {((IResponse)context.Variables["introspectToken"]).Body.As<JObject>()["jwt"]}")</value>
                </set-header>
            </otherwise>
        </choose>
        <base /
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>













