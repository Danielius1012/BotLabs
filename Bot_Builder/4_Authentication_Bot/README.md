# Authentication Bot - Bot Builder

In the course of this lab, we will build a bot that can post anything on Facebook, but only after it gets the required permissions from the user via login. This example uses some code from the [Microsoft Bot Builder repository](https://github.com/Microsoft/BotBuilder/tree/master/CSharp/Samples/SimpleFacebookAuthBot).

We will not cover how to publish or register your bot. To learn more about publishing and registration, take a look at the [Basic Echo Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/1_Basic_Echo_Bot).

## Setting up the Authentication bot ##

1. **[Optional]** If you don't know how to create a bot project in Visual Studio, take a look at the [Basic Echo Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/1_Basic_Echo_Bot) for instructions how to do this and come back after finishing the "Setting up the bot" chapter.

1. Create a new project called **FacebookAuthBot** using the echo bot template.

1. To create a functional authentication and the ability to post, we need 3 components:
    1. A FacebookAuthDialog.cs -  to create the user interaction
    1. A FacebookHelper.cs - to wrap the functionality for easier use
    1. An OAuthCallbackController.cs - to handle the response from the Facebook authentication 

## Writing the Facebook Dialog ##

To interact with the user, a Facebook Dialog is needed. Therfore, we create a **Dialogs folder** and include the class **FacebookAuthDialog.cs** inside the newly created folder. Then we connect the Dialog with the **MessagesController.cs** inside the **Controllers** folder. 

```csharp
using System.Net.Http;
using System.Threading.Tasks;
using System.Web.Http;
using Microsoft.Bot.Connector;
using Microsoft.Bot.Builder.Dialogs;
using FacebookAuth.Dialogs;

namespace FacebookAuth
{
    [BotAuthentication]
    public class MessagesController : ApiController
    {
        /// <summary>
        /// POST: api/Messages
        /// Receive a message from a user and reply to it
        /// </summary>
        public async Task<HttpResponseMessage> Post([FromBody]Activity activity)
        {
            if (activity.Type == ActivityTypes.Message)
            {
                await Conversation.SendAsync(activity, () => FacebookAuthDialog.dialog);
            }
            else
            {
                HandleSystemMessage(activity);
            }
            return new HttpResponseMessage(System.Net.HttpStatusCode.Accepted);
        }

        private Activity HandleSystemMessage(Activity message)
        {
            if (message.Type == ActivityTypes.DeleteUserData)
            {
                // Implement user deletion here
                // If we handle user deletion, return a real message
            }
            else if (message.Type == ActivityTypes.ConversationUpdate)
            {
                // Handle conversation state changes, like members being added and removed
                // Use Activity.MembersAdded and Activity.MembersRemoved and Activity.Action for info
                // Not available in all channels
            }
            else if (message.Type == ActivityTypes.ContactRelationUpdate)
            {
                // Handle add/remove from contact lists
                // Activity.From + Activity.Action represent what happened
            }
            else if (message.Type == ActivityTypes.Typing)
            {
                // Handle knowing tha the user is typing
            }
            else if (message.Type == ActivityTypes.Ping)
            {
            }

            return null;
        }
    }
}
```

After the connection, we will build the dialog itself. This will be a really simple dialog that uses regular expressions to differentiate between the commands: login, logout, post and delete.

Therefore we use the IDialog interface for implementation. The first step includes the integration of the StartAsync method needed for the dialog:

```csharp
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Connector;
using System;
using System.Text.RegularExpressions;
using System.Threading.Tasks;
using static FacebookAuth.Helper.FacebookHelper;

namespace FacebookAuth.Dialogs
{
    [Serializable]
    public class FacebookAuthDialog : IDialog<string>
    {
        public async Task StartAsync(IDialogContext context)
        {
            await LogIn(context);
        }

        
    }
}
```

Afterwards, we included necessary values and the constructor to the dialog:

```csharp
public static readonly Uri FacebookOauthCallback = new Uri("http://localhost:3979/api/OAuthCallback");

public static readonly string AuthTokenKey = "AuthToken";

public readonly ResumptionCookie ResumptionCookie;

public FacebookAuthDialog(IMessageActivity msg)
{
    ResumptionCookie = new ResumptionCookie(msg);
}
```

The **FacebookOauthCallback** is the endpoint created later in this lab, that is called, when the login is done. After deployment, this should be changed into the url of the website, with the addition of "/api/OAuthCallback".

To perform the login, we include the **login** method, which is called every time a user posts a message. If the user is already logged in, the AuthToken already exists and doesn't need to be requested again:

```csharp
private async Task LogIn(IDialogContext context)
{
    string token;
    if (!context.PrivateConversationData.TryGetValue(AuthTokenKey, out token))
    {
        context.PrivateConversationData.SetValue("persistedCookie", ResumptionCookie);

        // sending the sigin card with Facebook login url
        var reply = context.MakeMessage();
        var fbLoginUrl = FacebookHelpers.GetFacebookLoginURL(ResumptionCookie, FacebookOauthCallback.ToString());
        reply.Text = "Please login in using this card";
        reply.Attachments.Add(SigninCard.Create("You need to authorize me",
                                                "Login to Facebook!",
                                                fbLoginUrl
                                                ).ToAttachment());
        await context.PostAsync(reply);
        context.Wait(MessageReceivedAsync);
    }
    else
    {
        context.Done(token);
    }
}
```

The token is handled and stored by the **MessageReceivedAsync** method:

```csharp
private async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> argument)
{
    var msg = await (argument);
    if (msg.Text.StartsWith("token:"))
    {
        // Dialog is resumed by the OAuth callback and access token
        // is encoded in the message.Text
        var token = msg.Text.Remove(0, "token:".Length);
        context.PrivateConversationData.SetValue(AuthTokenKey, token);
        context.Done(token);
    }
    else
    {
        await LogIn(context);
    }
}
```

Finally, we include the actual dialog itself, which includes a so-called **Chain**. The chain includes a switch function, which takes in a number of cases. All of these cases are checked everytime a user writes a message. The Regex in every case is performed and if one of them evaluates as true, its function gets called. If not, the DefaultCase funtion is performed.

```csharp
public static readonly IDialog<string> dialog = Chain
    .PostToChain()
    .Switch(
        new Case<IMessageActivity, IDialog<string>>((msg) =>
        {
            var regex = new Regex("^login", RegexOptions.IgnoreCase);
            return regex.IsMatch(msg.Text);
        }, (ctx, msg) =>
        {
                    // Login Flow: https://developers.facebook.com/docs/facebook-login/manually-build-a-login-flow#confirm
                    // User wants to login, send the message to Facebook Auth Dialog
                    return Chain.ContinueWith(new FacebookAuthDialog(msg),
                        async (context, res) =>
                        {
                                    // The Facebook Auth Dialog completed successfully and returend the access token in its results
                                    var token = await res;
                            var valid = await FacebookHelpers.ValidateAccessToken(token);
                            var name = await FacebookHelpers.GetFacebookProfileName(token);
                            context.UserData.SetValue("name", name);
                            return Chain.Return($"Your are logged in as: {name}");
                        });
        }),
        new Case<IMessageActivity, IDialog<string>>((msg) =>
        {
            var regex = new Regex("^logout", RegexOptions.IgnoreCase);
            return regex.IsMatch(msg.Text);
        }, (ctx, msg) =>
        {
                    // Clearing user related data upon logout
                    ctx.PrivateConversationData.RemoveValue(AuthTokenKey);
            ctx.UserData.RemoveValue("name");
            return Chain.Return($"Your are logged out!");
        }),
        new Case<IMessageActivity, IDialog<string>>((msg) =>
        {
            var regex = new Regex("^post", RegexOptions.IgnoreCase);
            return regex.IsMatch(msg.Text);
        }, (ctx, msg) =>
        {
                    // Post the message on Facebook
                    // User wants to login, send the message to Facebook Auth Dialog
                    string postContent = msg.Text.Substring(5);
            return Chain.ContinueWith(new FacebookAuthDialog(msg),
                        async (context, res) =>
                        {
                                    // The Facebook Auth Dialog completed successfully and returend the access token in its results
                                    var token = await res;
                            var valid = await FacebookHelpers.ValidateAccessToken(token);
                            var id = await FacebookHelpers.GetFacebookProfileId(token);
                            await FacebookHelpers.PostMessageToFacebook(postContent, token, id);
                            return Chain.Return($"Message was successfully posted");
                        });
        }),
        new Case<IMessageActivity, IDialog<string>>((msg) =>
        {
            var regex = new Regex("^delete", RegexOptions.IgnoreCase);
            return regex.IsMatch(msg.Text);
        }, (ctx, msg) =>
        {
                    // Delete the app permissions
                    return Chain.ContinueWith(new FacebookAuthDialog(msg),
                        async (context, res) =>
                        {
                                    // The Facebook Auth Dialog completed successfully and returend the access token in its results
                                    var token = await res;
                            var valid = await FacebookHelpers.ValidateAccessToken(token);
                            var id = await FacebookHelpers.GetFacebookProfileId(token);
                            await FacebookHelpers.DeletePermissions(token, id);
                            ctx.PrivateConversationData.RemoveValue(AuthTokenKey);
                            ctx.UserData.RemoveValue("name");
                            return Chain.Return($"Permissions deleted");
                        });
        }),
        new DefaultCase<IMessageActivity, IDialog<string>>((ctx, msg) =>
        {
            string token;
            string name = string.Empty;
            if (ctx.PrivateConversationData.TryGetValue(AuthTokenKey, out token) && ctx.UserData.TryGetValue("name", out name))
            {
                var validationTask = FacebookHelpers.ValidateAccessToken(token);
                validationTask.Wait();
                if (validationTask.IsCompleted && validationTask.Result)
                {
                    return Chain.Return($"Your are logged in as: {name}");
                }
                else
                {
                    return Chain.Return($"Your Token has expired! Say \"login\" to log you back in!");
                }
            }
            else
            {
                return Chain.Return("Say \"login\" when you want to login to Facebook!");
            }
        })
    ).Unwrap().PostToUser();
```

## Wrapping the Facebook functionality with FacebookHelper ##

Until now, we implemented the interaction with the user but not actual functionality. This changes with the **FacebookHelper.cs**. This file is created inside a new folder called **Helper**.

So let's start off by implementing two classes, that save important information. These are **FacebookAcessToken** and **FacebookProfile**:

```csharp
public class FacebookAcessToken
{
    public FacebookAcessToken()
    {
    }

    [JsonProperty(PropertyName = "access_token")]
    public string AccessToken { get; set; }

    [JsonProperty(PropertyName = "token_type")]
    public string TokenType { get; set; }

    [JsonProperty(PropertyName = "expires_in")]
    public long ExpiresIn { get; set; }
}

class FacebookProfile
{
    public FacebookProfile()
    {
    }

    [JsonProperty(PropertyName = "id")]
    public string Id { get; set; }
    [JsonProperty(PropertyName = "name")]
    public string Name { get; set; }
}
```

The actual FacebookHelpers are setup by creating the following class:

```csharp
public static class FacebookHelpers
{
    // The Facebook App Id
    public static readonly string FacebookAppId = "<INSERT APP ID HERE>";

    // The Facebook App Secret
    public static readonly string FacebookAppSecret = "<INSERT APP SECRET HERE>";
}
```

The values for app id and secret can be obtained by [creating a new Facebook App](https://developers.facebook.com/apps).

Now the bot has the permission to post any message you give to it on your behalf. If you don't want the permissions to stay, you can delete them completely with the DeletePermissions method. The [Facebook guide on deleting permissions](https://developers.facebook.com/docs/facebook-login/permissions/requesting-and-revoking) is not quite clear on what is exactly needed to delete the permissions. These are 2 things: 
- a DELETE call to  https://graph.facebook.com/{user-id}/permissions
- a URL parameter with the token current app token "access_token"

The final delete method looks as follows:

```csharp
public static async Task DeletePermissions(string accessToken, string id)
{
    // Delete Permissions: https://developers.facebook.com/docs/facebook-login/permissions/requesting-and-revoking
    var uri = GetUri($"https://graph.facebook.com/{id}/permissions",
        Tuple.Create("access_token", accessToken));

    using (HttpClient client = new HttpClient())
    {
        var a = await client.DeleteAsync(uri.ToString());
    }
}
```

## Creating the OAuth Controller ##

...

## Testing the service ##

We created 4 tasks the bot can perform: login, logout, post a message and delete permissions.

## Further Steps ##

1. To learn about other bots, try out the tutorials on [Form Flow Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/2_Form_Flow_Bot) or [LUIS Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/3_LUIS_Bot)

1. If you want to get to know a even more agile and Cloud optimized way to develop a bot, check out the labs on the [Azure Bot Service](https://github.com/Danielius1012/BotLabs/tree/master/Azure_Bot_Service)