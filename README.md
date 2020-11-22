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

Complete Policy defined for our API manager is given in policy.xml


\
