# ToDo
Code clean:
* morgan: Can be removed. As pino logger has been used for logging.  
* Middleware:  
 server.ts -> configureMiddleware(). 
    - Compression: improve Compression logic only the requests greater than 5kb to improve performance.  
    - /health: it should be **GET** method rather than middleware function(app.use).  
    - Static route check: apiWhiteListLogger() method is already checking for static route for app.all(*). Again the same check is happening for app.all(*, isAllowed()). In isAllowed() methos showuldAllow() check in if condition can be removed as it is duplicate check. All this already checking in apiWhiteListLogger() method.
    - code clean: '/resource', '/eclogin' can be added directly to the checkIsStaticRoute() -> excludePath list. 
    the below if condition can be removed fully in apiWhiteList.ts -> isAllowed() method. We should check for isStatic then allow.
    ```
    if (shouldAllow(req) || _.includes(REQ_URL, '/resource') || _.includes(REQ_URL, '/eclogin')) {
                logDebug('Path : ' + REQ_URL + ' is in excluded list.')
                next()
            }
    ```
    - fileupload - Use Busboy properties to set the limit of max filesize & use temp store instead of memory store for upload file process.  
    server.ts => configureMiddleware() => this.app.use(fileUpload());  
    https://www.npmjs.com/package/express-fileupload. 
* keycloak
    - authApi also should be keycloak.protect but it's not. What is the reason?
    ```
    private authoringApi() {
        if (this.keycloak) {
        this.app.use('/authSearchApi', this.keycloak.protect, authSearch)
        this.app.use('/authApi', authApi)
        }
    }
    ```



## Configurations not in use
* APP_CONFIGURATIONS
* APP_LOGS

