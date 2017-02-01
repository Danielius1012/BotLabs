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

To interact with the user, a Facebook Dialog is needed. Therfore, we create 

## Wrapping the Facebook functionality with FacebookHelper ##

...

Now the bot has the permission to post any message you give to it on your behalf, but if you don't want the permissions to stay, you can delete them completely with the DeletePermissions method. The [Facebook guide on deleting permissions](https://developers.facebook.com/docs/facebook-login/permissions/requesting-and-revoking) is not quite clear on what is exactly needed to delete the permissions. These are 2 things: 
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