# URLMock

URLMock is an Objective-C framework for mocking and stubbing URL requests and
responses. It works with APIs built on the Foundation NSURL loading
system—NSURLConnection, NSURLSession, and AFNetworking, for example—with almost
no changes to your code.


## Features

* Simple, well-documented Objective-C API
* Minimal setup necessary
* Works with APIs built atop the Foundation NSURL loading system
* Designed with both response stubbing and unit testing in mind
* Can be used for some or all of your project’s URL requests
* Well-tested and includes lots of helpful testing utilities
* Works on both Mac OS X and iOS


## What’s New in URLMock 1.2.1

URLMock 1.2.1 adds support for iOS 6.0. URLMock 1.2 is a major update that adds
the following new functionality:

* Pattern-based request matching using `UMKPatternMatchingMockRequest`
* Support for stream-based requests, including those created with `NSURLSession`
* More complete support for Rails/Rack-style nested parameter query strings.
  We should now be able to encode and decode query strings with deeply nested
  objects, e.g., strings inside dictionaries inside arrays inside dictionaries
  inside arrays inside dictionaries.
* New test utility functions for generating random errors and C identifier
  strings
* Official support for OS X 10.8 and iOS 6.0.


## Installation

The easiest way to start using URLMock is to install it with CocoaPods. 

    pod 'URLMock', '~> 1.2.1'

You can also build it and include the built products in your project. For OS X,
just add `URLMock.framework` to your project. For iOS, add URLMock’s public
headers to your header search path and link in `libURLMock.a`.

Note that URLMock depends on [SOCKit][SOCKit], so it may be necessary to get
that separately if you’re not using CocoaPods. That said, we highly recommend
you use CocoaPods.

[SOCKit]: https://github.com/NimbusKit/sockit "SOCKit"

### Installing Subspecs

URLMock has two CocoaPods subspecs, `TestHelpers` and `SubclassResponsibility`. 
`TestHelpers` includes a wide variety of useful testing functions. See
[`UMKTestUtilities.h`][UMKTestUtilities] for more details. It can be installed by
adding the following line to your Podfile:

    pod 'URLMock/TestHelpers', '~> 1.2.1'

Similarly, the `SubclassResponsibility` subspec can be installed by adding the 
following line to your Podfile:

    pod 'URLMock/SubclassResponsibility', '~> 1.2.1'

This subspec adds methods to `NSException` to easily raise exceptions in methods 
for which subclasses must provide an implementation. See 
[NSException+UMKSubclassResponsibility.h][SubclassResponsibility] for details.

[UMKTestUtilities]: https://github.com/twotoasters/URLMock/blob/master/URLMock/Utilities/UMKTestUtilities.h "UMKTestUtilities.h"
[SubclassResponsibility]: https://github.com/twotoasters/URLMock/blob/master/URLMock/Categories/NSException%2BUMKSubclassResponsibility.h "NSException+UMKSubclassResponsibility.h"


## Using URLMock

URLMock is designed with both response stubbing and unit testing in mind. Both
work very similarly.

### Response stubbing

Using URLMock for response stubbing is simple:

1. Enable URLMock.

        [UMKMockURLProtocol enable];

   If you are using `NSURLSession` and not using the shared session, you also
   need to add `UMKMockURLProtocol` to your session configuration’s set of
   allowed protocol classes.

        NSURLSessionConfiguration *configuration = …;
        configuration.protocolClasses = @[ [UMKMockURLProtocol class] ];
        NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration];

2. Add an expected mock request and response.

        // The request is a POST with some JSON data
        NSURL *URL = [NSURL URLWithString:@"http://host.com/api/v1/person"];
        id requestJSON = @{ @"person" : @{ @"name" : @"John Doe", 
                                           @"age" : @47 } };
        id responseJSON = @{ @"person" : @{ @"id" : @1, 
                                            @"name" : @"John Doe", 
                                            @"age" : @47 } };

        [UMKMockURLProtocol expectMockHTTPPostRequestWithURL:URL 
                                                 requestJSON:requestJSON
                                          responseStatusCode:200
                                                responseJSON:responseJSON];
   
   Mock requests and responses are not limited to having JSON bodies; they can 
   also have bodies with strings, WWW form-encoded parameter dictionaries, or
   arbitrary NSData instances. There are also mock responders for responding
   with an error or returning data in chunks with a delay between each chunk,
   and we’ll be adding more responders in the future.

3. Execute your real request to get the stubbed response back. You don’t have to 
   make any changes to your code when using URLMock. Things should just work. 
   For example, the following URLConnection code will receive the mock response 
   above:
   
        NSURL *URL = [NSURL URLWithString:@"http://host.com/api/v1/person"];
        NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:URL];
        request.HTTPMethod = @"POST";
        id bodyJSON = @{ @"person" : @{ @"name" : @"John Doe", @"age" : @47 } };
        request.HTTPBody = [NSJSONSerialization dataWithJSONObject:bodyJSON
                                                           options:0 
                                                             error:NULL];
        
        // Create the connection as usual
        NSURLConnection *connection = [[NSURLConnection alloc] initWithRequest:request delegate:…];
   
   
   The following AFNetworking code would accomplish the same thing:
   
        NSURL *base = [NSURL URLWithString:@"http://host.com/api/v1/"];
        id params = @{ @"person" : @{ @"name" : @"John Doe", @"age" : @47 } };

        // Send a POST as usual
        AFHTTPRequestOperationManager *om = [[AFHTTPRequestOperationManager alloc] initWithBaseURL:base];
        om.requestSerializer = [AFJSONRequestSerializer serializer];
        [om POST:@"person" parameters:params success:^(AFHTTPRequestOperation *op, id object) {
            …
        } failure:^(AFHTTPRequestOperation *op, NSError *error) {
            …
        }];    


### Pattern-Matching Mock Requests

You can also create a mock request that responds dynamically to the request it
matches against using `UMKPatternMatchingMockRequest`. To create a
pattern-matching mock request, you need to provide a URL pattern, e.g.,
`@"http://hostname.com/:resource/:resourceID"`. When a URL request matches this
pattern, the mock request generates an appropriate responder using its responder
generation block:

        NSString *pattern = @"http://hostname.com/accounts/:accountID/followers";
        UMKPatternMatchingMockRequest *mockRequest =  [[UMKPatternMatchingMockRequest alloc] initWithPattern:pattern];
        mockRequest.HTTPMethods = [NSSet setWithObject:kUMKMockHTTPRequestPostMethod];
        
        mockRequest.responderGenerationBlock = ^id<UMKMockURLResponder>(NSURLRequest *request, NSDictionary *parameters) {
            NSDictionary *requestJSON = [request umk_JSONObjectFromHTTPBody];

            // Respond with 
            //   { 
            //     "follower_id": «New follower’s ID»,
            //     "following_id":  «Account ID that was POSTed to» 
            //   }
            UMKMockHTTPResponder *responder = [UMKMockHTTPResponder mockHTTPResponderWithStatusCode:200];
            [responder setBodyWithJSONObject:@{ @"follower_id" : requestJSON[@"follower_id"] 
                                                @"following_id" : @([parameters[@"accountID"] integerValue]) }];
            return responder;
        };
        
        [UMKMockURLProtocol addExpectedMockRequest:mockRequest];

See the documentation for `UMKPatternMatchingMockRequest` for more information.


### Unit testing

Unit testing with URLMock is very similar to response stubbing, but you can use 
a few additional APIs to make unit testing easier.

1. Enable verification in UMKMockURLProtocol using `+setVerificationEnabled:`. 
   This enables tracking whether any unexpected requests were received. It makes
   sense to do this in your XCTestCase’s `+setUp` method. You can also disable 
   verification in your `+tearDown` method.
   
        + (void)setUp
        {
            [super setUp];
            [UMKMockURLProtocol enable];
            [UMKMockURLProtocol setVerificationEnabled:YES];
        }
        
        + (void)tearDown
        {
            [UMKMockURLProtocol setVerificationEnabled:NO];
            [UMKMockURLProtocol disable];
            [super tearDown];
        }
   
2. Before (or after) each test, invoke `+[UMKMockURLProtocol reset]`. This 
   resets UMKMockURLProtocol’s expectations to their original state. It does not
   change whether verification is enabled. 
   
   If you’re using XCTest, the ideal place to do this is in your test case’s
   `-setUp` (or `-tearDown`) method.
   
        - (void)setUp
        {
           [super setUp];
           [UMKMockURLProtocol reset];
        }


3. After you’ve executed the code you’re testing, send UMKMockURLProtocol the 
   `+verifyWithError:` message. It will return YES if all expected mock requests
   were serviced and no unexpected mock requests were received.

        NSError *error = nil;
        XCTAssertTrue([UMKMockURLProtocol verifyWithError:&error], @"…");

4. For the strictest testing, enable header checking on your UMKMockHTTPRequest 
   instances. When enabled, mock requests only match URL requests that have
   equivalent headers. You can enable header checking on mock HTTP request by
   setting its `checksHeadersWhenMatching` to YES or by using
   `initWithHTTPMethod:URL:checksHeadersWhenMatching:`.
   
        UMKMockHTTPRequest *request = [UMKMockHTTPRequest mockHTTPGetRequestWithURL:URL];
        request.checksHeadersWhenMatching = YES;
   
   Note that some networking APIs—most notably AFNetworking—send headers that
   you didn’t explicitly set, so you should determine what those are before
   creating your mock requests. To make things a little easier, you can use
   `+[UMKMockHTTPRequest setDefaultHeaders:]` to set the default headers for new
   UMKMockHTTPRequest instances. For example, if you’re using AFNetworking’s 
   default HTTP request serializer, you can set default headers this way:
   
        [UMKMockHTTPRequest setDefaultHeaders:[[AFHTTPRequestSerializer serializer] HTTPRequestHeaders]];


### Non-HTTP Protocols

Out of the box, URLMock only supports HTTP and HTTPS. However, it is designed to
work with any URL protocol that the NSURL loading system can support. If you are
using a custom scheme or URL protocol and would like to add support for mocking
requests and responses, you need only create classes that conform to the
`UMKMockURLRequest` and `UMKMockURLResponder` protocols. See the implementations
of `UMKMockHTTPRequest` and `UMKMockHTTPResponder` for examples.


## Contributing, Filing Bugs, and Requesting Enhancements

URLMock is very usable in its current state, but there’s still a lot that could
be done. If you would like to help fix bugs or add features, send us a pull 
request!

We use GitHub issues for bugs, enhancement requests, and the limited support we
provide, so open an issue for any of those.

 
## License

All code is licensed under the MIT license. Do with it as you will.
