---
layout: post
title:  "DotNetOpenAuth, OAuth, and MVC For Dummies"
---

I recently was trying to understand OAuth so that I could utilize the LinkedIn API in my Asp.Net MVC application.  The <a href="https://developer.linkedin.com/documents/oauth-overview">LinkedIn API</a> has a pretty good summary of the OAuth process.  However, the more I looked at it (and other documentation on OAuth), the more confused I got at implementing OAuth myself.  So I went around and looked for a C# library to help me out, and found <a href="http://www.dotnetopenauth.net/">DotNetOpenAuth.Net</a>.  Unfortunately, the DotNetOpenAuth website is horribly designed without any real tutorials.  After searching around the internet I was able to piece some things together, and hopefully this will help someone figure this out quicker than I was able to.

<h2>Requesting User Authorization</h2>

The first step of authorizing with OAuth and DotNetOpenAuth is to redirect to the OAuth provider's authorization page, so the user can grant your application access to perform queries/service calls on their behalf.  DotNetOpenAuth needs several pieces of information to begin this process.  The first is to create a ServiceProviderDescription object, that contains the provider's URL for retrieving the request token, URL for retrieving the access token, URL for requesting user authentication,  what OAuth protocol version to use, and details of the tamper protection used to encode the OAuth signature.  An example of creating the provider description for connecting to LinkedIn is:

{% highlight csharp %}
private ServiceProviderDescription GetServiceDescription()
{
    return new ServiceProviderDescription
    {
        AccessTokenEndpoint = new MessageReceivingEndpoint("https://api.linkedin.com/uas/oauth/accessToken", HttpDeliveryMethods.PostRequest),
        RequestTokenEndpoint = new MessageReceivingEndpoint("https://api.linkedin.com/uas/oauth/requestToken", HttpDeliveryMethods.PostRequest),
        UserAuthorizationEndpoint = new MessageReceivingEndpoint("https://www.linkedin.com/uas/oauth/authorize", HttpDeliveryMethods.PostRequest),
        TamperProtectionElements = new ITamperProtectionChannelBindingElement[] { new HmacSha1SigningBindingElement() },
        ProtocolVersion = ProtocolVersion.V10a
    };
}
{% endhighlight %}

The next thing that DotNetOpenAuth requires is a token manager.  The token manager is a class which DotNetOpenAuth utilizes to store and retrieve the consumer key, consumer secret, and a token secret for a given access key.  Since how you will store the user access tokens and token secrets will vary project to project, DotNetOpenAuth assumes you will create your own token storage and retrieval mechanism by implementing the IConsumerTokenManager interface.

For testing, I looked online for an in memory token manager class, and found the following code:

{% highlight csharp %}
public class InMemoryTokenManager : IConsumerTokenManager, IOpenIdOAuthTokenManager
{
    private Dictionary<string, string> tokensAndSecrets = new Dictionary<string, string>();

    public InMemoryTokenManager(string consumerKey, string consumerSecret)
    {
        if (String.IsNullOrEmpty(consumerKey))
        {
            throw new ArgumentNullException("consumerKey");
        }

        this.ConsumerKey = consumerKey;
        this.ConsumerSecret = consumerSecret;
    }

    public string ConsumerKey { get; private set; }

    public string ConsumerSecret { get; private set; }

    #region ITokenManager Members

    public string GetConsumerSecret(string consumerKey)
    {
        if (consumerKey == this.ConsumerKey)
        {
            return this.ConsumerSecret;
        }
        else
        {
            throw new ArgumentException("Unrecognized consumer key.", "consumerKey");
        }
    }

    public string GetTokenSecret(string token)
    {
        return this.tokensAndSecrets[token];
    }

    public void StoreNewRequestToken(UnauthorizedTokenRequest request, ITokenSecretContainingMessage response)
    {
        this.tokensAndSecrets[response.Token] = response.TokenSecret;
    }

    public void ExpireRequestTokenAndStoreNewAccessToken(string consumerKey, string requestToken, string accessToken, string accessTokenSecret)
    {
        this.tokensAndSecrets.Remove(requestToken);
        this.tokensAndSecrets[accessToken] = accessTokenSecret;
    }

    /// <summary>
    /// Classifies a token as a request token or an access token.
    /// </summary>
    /// <param name="token">The token to classify.</param>
    /// <returns>Request or Access token, or invalid if the token is not recognized.</returns>
    public TokenType GetTokenType(string token)
    {
        throw new NotImplementedException();
    }

    #endregion

    #region IOpenIdOAuthTokenManager Members

    public void StoreOpenIdAuthorizedRequestToken(string consumerKey, AuthorizationApprovedResponse authorization)
    {
        this.tokensAndSecrets[authorization.RequestToken] = string.Empty;
    }

    #endregion
}
{% endhighlight %}

Now that we have a Token Manager class to use, and a service description we can begin the authorization process.  This can be accomplished with the following code:

{% highlight csharp %}
public ActionResult StartOAuth()
{
    var serviceProvider = GetServiceDescription();
    var consumer = new WebConsumer(serviceProvider, _tokenManager);

    // Url to redirect to
    var authUrl = new Uri(Request.Url.Scheme + "://" + Request.Url.Authority + "/Home/OAuthCallBack");

    // request access
    consumer.Channel.Send(consumer.PrepareRequestUserAuthorization(authUrl, null, null));

    // This will not get hit!
    return null;
}
{% endhighlight %}

This sets up the DotNetOpenAuth consumer object to use our in memory token manager, and our previously defined service description object.  We then form the URL we want the service provider to redirect to after the user grants your application access.  Finally we tell the consumer to send the user authorization request.  The Send() method will end the execution of the Asp.Net page, and thus no code after the Send() call will be called.  The user will then see the authorization page on the service provider's website, which will allow them to allow or deny access for your application.  

<h2>Receiving the OAuth CallBack</h2>

Once the user logs into the service provider and gives your application authorization, the service provider will redirect to the callback URL specified in the previous code.  The service provider includes the oauth token and secret, which needs to be processed by DotNetOpenAuth.  The following code can be done to process the auth token and store it and the secret in the token manager:

{% highlight csharp %}
public ActionResult OAuthCallback()
{
    // Process result from the service provider
    var serviceProvider = GetServiceDescription();
    var consumer = new WebConsumer(serviceProvider, _tokenManager);
    var accessTokenResponse = consumer.ProcessUserAuthorization();

    // If we didn't have an access token response, this wasn't called by the service provider
    if (accessTokenResponse == null)
        return RedirectToAction("Index");

    // Extract the access token
    string accessToken = accessTokenResponse.AccessToken;

    ViewBag.Token = accessToken;
    ViewBag.Secret = _tokenManager.GetTokenSecret(accessToken);
    return View();
}
{% endhighlight %}

<h2>Perform A Request Using OAuth Credentials</h2>

Now that we have the user's authorization details we can perform API queries.  In order to query the API though, we need to sign our requests with a combination of the user's access token and our consumer key.  Since we retrieved the user's access token in the previous code, you need to figure out a way to store that somewhere, either in the user's record in the database, in a cookie, or any other way you can quickly get at it again without requiring the user to constantly re-auth.

In order to use that access token to call a service provider API function, you can form a prepared HttpWebRequest by calling the PrepareAuthorizedRequest() method on the WebConsumer class.  The following is an example how to use an access token to query the LinkedIn API.

{% highlight csharp %}
public ActionResult Test2()
{
    // Process result from linked in
    var LiServiceProvider = GetServiceDescription();
    var linkedIn = new WebConsumer(LiServiceProvider, _tokenManager);
    var accessToken = GetAccessTokenForUser();

    // Retrieve the user's profile information
    var endpoint = new MessageReceivingEndpoint("http://api.linkedin.com/v1/people/~", HttpDeliveryMethods.GetRequest);
    var request = linkedIn.PrepareAuthorizedRequest(endpoint, accessToken);
    var response = request.GetResponse();
    ViewBag.Result = (new StreamReader(response.GetResponseStream())).ReadToEnd();

    return View();
}
{% endhighlight %}

And now, if the user has authenticated going to /Home/Test2 will correctly access LinkedIn on behalf of the user!

<b>Update:</b> For those who are looking for a tool to help test API calls prior to having to write them down formally in code, please see my [Testing Oauth APIs Without Coding]({{ site.baseurl }}{% post_url 2011-12-21-testing-oauth-apis-without-coding %}) article!