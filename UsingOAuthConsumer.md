## Introduction ##

This guide is for developers interested in adding support for [OAuth](http://oauth.net) to their Mac apps. In order to use the framework, you must first get a copy from the [svn repository](http://oauth.googlecode.com/svn/code/obj-c/) and compile it. Compiling the framework will automatically run all of the unit tests.

## Importing the Framework ##

There are 3 steps for adding the pre-compiled framework to your app in Xcode:

  1. Drag the framework into your Linked Frameworks group in Xcode, choosing to copy the framework instead of simply referencing it.
  1. Create a new Copy Files Build Phase for your app's main target.
  1. Drag the framework from the Linked Frameworks folder into the new Copy Files Build Phase and select "Frameworks" as its destination.

**For now, please use this as a private embedded framework rather than copying it to the system's framework folder upon installation.**

## Using OAuthConsumer ##

### Set Up ###

The first thing you will need in order to access a service providing OAuth support is a Consumer Key and a Consumer Secret. These must be obtained directly from the service provider. The OAuth Core 1.0 Spec does not address automatically generating such items.

The second thing required is a set of URLs provided for OAuth interactions by your service provider. These should be documented by the service provider and include:

  * Request Token URL
  * User Authorization URL
  * Access Token URL

Additionally, there should be a set of API standards to use in order to interact with the service after obtaining the OAuth credentials.

Once you have obtained your Consumer Key, Consumer Secret, and know your OAuth endpoint URLs, you are ready to begin writing code.

### Getting an Unauthorized Request Token ###

The first order of business when communicating with an OAuth service provider is obtaining an Unauthorized Request Token. Using your Consumer Key, Consumer Secret, and the service provider's Request Token URL, you can send a request for an Unauthorized Request Token like this:

```

    OAConsumer *consumer = [[OAConsumer alloc] initWithKey:@"mykey"
                                                    secret:@"mysecret"];

    NSURL *url = [NSURL URLWithString:@"http://example.com/get_request_token"];

    OAMutableURLRequest *request = [[OAMutableURLRequest alloc] initWithURL:url
                                                                   consumer:consumer
                                                                      token:nil   // we don't have a Token yet
                                                                      realm:nil   // our service provider doesn't specify a realm
                                                          signatureProvider:nil]; // use the default method, HMAC-SHA1

    [request setHTTPMethod:@"POST"];

    OADataFetcher *fetcher = [[OADataFetcher alloc] init];

    [fetcher fetchDataWithRequest:request
                         delegate:self
                didFinishSelector:@selector(requestTokenTicket:didFinishWithData:)
                  didFailSelector:@selector(requestTokenTicket:didFailWithError:)];

```

Here is an example of a delegate method that uses a successful response to create the Request Token:

```

    - (void)requestTokenTicket:(OAServiceTicket *)ticket didFinishWithData:(NSData *)data {
        if (ticket.didSucceed) {
            NSString *responseBody = [[NSString alloc] initWithData:data
                                                           encoding:NSUTF8StringEncoding];
            requestToken = [[OAToken alloc] initWithHTTPResponseBody:responseBody];
        }
    }

```

### Authorizing the Request Token ###

Now that you have an unauthorized request token, direct your user to the service provider's User Authorization URL so that they can approve access for your application:

```

    NSURL *url = [NSURL URLWithString:@"http://example.com/authorize"];
    [[NSWorkspace sharedWorkspace] openURL:url];

```

The service provider should first ensure the user is logged in, requiring authentication if this is not the case. The user will then have an opportunity to grant access for your application on the service provider's site. Service providers can use this interaction to limit access to certain kinds of assets if desired.

### Obtaining an Access Token ###

Now that the Request Token has been authorized, an Access Token can be obtained in the same manner as described in "Getting an Unauthorized Request Token" with two exceptions:

  1. Instead of passing `nil` for the token parameter when creating the `OAMutableURLRequest`, pass the token obtained with the `requestTokenTicket:didFinishWithData:` method, above.
  1. The delegate methods might choose different names such as `accessTokenTicket:didFinishWithData:` and `accessTokenTicket:didFailWithError:`.

The delegate method can create the Access Token as an `OAToken` object in exactly the same manner as described above for the Request Token.

### Using the Keychain ###

If you choose to store this token in the user's Keychain, it can be done like this:

```

    [accessToken storeInDefaultKeychainWithAppName:@"MyApp"
                               serviceProviderName:@"Example.com"];

```

Retrieving that same token from the Keychain on a different run of your app looks like this:

```

    OAToken *accessToken = [[OAToken alloc] initWithKeychainUsingAppName:@"MyApp"
                                                     serviceProviderName:@"Example.com"];

```

### Accessing Protected Resources ###

At this point, you are ready to access the service provider's APIs, authenticating with OAuth in the background. You can either send requests using `OAMutableURLRequest` in the same manner as you would normally do with `NSMutableURLRequest` or you can continue to use the convenience objects `OAServiceTicket` and `OADataFetcher`. If you choose the former, be sure to call the `prepare` method on the request prior to initiating the connection or your request will not contain valid OAuth parameters.

Speaking of parameters, there are methods added to `NSMutableURLRequest` (and thus `OAMutableURLRequest`) to support setting parameters for requests. At the moment, there is a requirement to first set the HTTP method, then add or get the parameters as an `NSArray`.

Here is an example OAuth request for a protected resource using the GET method (default), HTTPS, the PLAINTEXT signature method, passing two parameters:

  1. title = My Page
  1. description = My Page Holds Text

```

    NSURL *url = [NSURL URLWithString:@"https://example.com/user/1/flights/"];
    OAMutableURLRequest *request = [[OAMutableURLRequest alloc] initWithURL:url
                                                                   consumer:consumer
                                                                      token:accessToken
                                                                      realm:nil
                                                          signatureProvider:[[OAPlaintextSignatureProvider alloc] init]];
    
    OARequestParameter *nameParam = [[OARequestParameter alloc] initWithName:@"title"
                                                                       value:@"My Page"];
    OARequestParameter *descParam = [[OARequestParameter alloc] initWithName:@"description"
                                                                       value:@"My Page Holds Text"];
    NSArray *params = [NSArray arrayWithObjects:nameParam, descParam, nil];
    [request setParameters:params];
    
    OADataFetcher *fetcher = [[OADataFetcher alloc] init];
    [fetcher fetchDataWithRequest:request
                         delegate:self
                didFinishSelector:@selector(apiTicket:didFinishWithData:)
                  didFailSelector:@selector(apiTicket:didFailWithError:)];

```