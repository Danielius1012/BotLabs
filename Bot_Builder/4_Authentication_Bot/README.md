# Authentification Bot - Bot Builder

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

...

## Wrapping the Facebook functionality with FacebookHelper ##

...

## Creating the OAuth Controller ##

...

## Testing the service ##

...

## Further Steps ##

1. To learn about other bots, try out the tutorials on [Form Flow Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/2_Form_Flow_Bot) or [LUIS Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/3_LUIS_Bot)

1. If you want to get to know a even more agile and Cloud optimized way to develop a bot, check out the labs on the [Azure Bot Service](https://github.com/Danielius1012/BotLabs/tree/master/Azure_Bot_Service)