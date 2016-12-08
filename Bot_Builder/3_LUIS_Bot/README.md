# LUIS Bot - Bot Builder

In this lab, we will create a bot that can learn and take over the world - more or less. What it can do though, is using the Machine Learning capabilities of the [LUIS Service](http://luis.ai) to learn phrases from the user and extract his/her intentions and entities used in the phrase. We will train it to be able to tell us the weather in any given city, using the API form [Open Weather Map](http://openweathermap.org/).

We will not cover how to publish or register your bot. To learn more about publishing and registration, take a look at the [Basic Echo Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/1_Basic_Echo_Bot).

## Setting up the LUIS bot ##

1. **[Optional]** If you don't know how to create a bot project in Visual Studio, take a look at the [Basic Echo Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/1_Basic_Echo_Bot) for instructions how to do this and come back after finishing the "Setting up the bot" chapter.

1. Create a new project called **WeatherBot** using the echo bot template.

1. To create a LUIS Bot from the template of the echo bot example, we have to send the message from the user to the LUIS service. Fortunately, the Bot Buider comes with LUIS support build in. To handle the service we create a new class, called **WeatherDialog.cs** in the **Controllers** directory.

    ![Add Weather Dialog](./_images/1_AddWeatherDialog.png)

    This dialog will be handling the message to LUIS and the response from the service. Every time LUIS sends a response, the intent will trigger a pre-defined method. Additionally, entities can be used to extract features from the user query. 

1. Replace the code in the newly created file with the code below to get started with the LUIS service:

    ```csharp
    using Microsoft.Bot.Builder.Dialogs;
    using Microsoft.Bot.Builder.Luis.Models;
    using System;
    using System.Threading.Tasks;

    namespace WeatherrBot.Controllers
    {
        [Serializable]
        public class WeatherDialog : LuisDialog<object>
        {
            [LuisIntent("")]
            public async Task None(IDialogContext context, LuisResult result)
            {
                string message = $"Sorry I did not understand: {result.Query}";
                await context.PostAsync(message);
                context.Wait(MessageReceived);
            }        
        }
    }
    ```

## Train the LUIS service ##

1. To hook the code up to the LUIS service, we have to create a service first. To do this, we go to the [LUIS website](https://www.luis.ai/) and log in with a Microsoft account.

    ![LUIS Login](./_images/2_LUISLogin.png)

1. In the following view called **My Applications** we create a new application, by clicking "New Application"

    ![New Application](./_images/3_AddNewApplication.png)

1. Fill out the information for the new service and click "Add App"

    ![Add Application](./_images/3_AddLUISApplication.png)

1. The service is then created and you are redirected to the model training screen:

    ![LUIS Overview](./_images/4_LUISOverview.png)

    For this lab we will concentrate on the intents and entities on the left side of the screen. 

1. We will now create a simple intent and entity. Lets start by adding an entity by clicking on the '+' next to it. We will call this entity **locationName** to get the location from the user's query.

    ![Add Entity](./_images/5_AddEntity.png)

1. **[Side Note]** To use already trained entity recognition, we can take advantage of the Pre-built entities. Just click on the '+' next to "Pre-built Entities" and pick geography.

    ![Add Pre built entity](./_images/6_AddPreBuiltEntity.png)

    We will not look at this option any further, because we want to get through the process of training the entity ourselves. It is very useful though, because it detects many different formats without training.

1. Afterwards, we create an intent to capture the intention of getting the weather for a specific location. A query can look like "What is the weather in London?". We create a new intent by clicking the '+' next to the Intents panel.

    ![Add Intent](./_images/7_AddIntent.png)

1. After this, the query is displayed in the middle of the page, but it doesn't use our entity yet. Let's change this by clicking the location name (in this case London) and choosing the entity 'locationName' and then click submit.

    ![Annotate Query](./_images/8_AnnotateQuery.png)

1. By inserting queries into the text field we can train the model to recognize other query formats and the already existing ones even better. Type 5 queries inside the text field (individually) and annotate them as shown before (highlighting the location, picking the intent from the dropdown). Click 'Submit' after every annotation.

1. After annotation, the model needs to be trained, which can be done by clicking 'Train' in the bottom left corner of the page.

    ![Train Model](./_images/9_TrainModel.png)

1. We can now publish the LUIS application by clicking 'Publish' right above the intents. 

    ![Publish Model](./_images/10_PublishModel_1.png)

    ![Publish Model](./_images/10_PublishModel_2.png)

1. After publishing, the url of the service will appear and you will be able to test your service with the displayed url. We need this query for later to authenticate our bot to communicate with the LUIS service.

1. Now the service is ready to detect our queries, so lets connect it to the Bot. **Note: Any time you add queries to the service, you need to re-train and re-publish your service for the changes to take effect.**
 
## Connect to the LUIS service ##

1. So let's get back to our bot and open the **WeatherDialog.cs** file. We have to add our credentials to connect the dialog to our LUIS application. Add the following 2 things:

    1. Import library

        ```csharp
        using Microsoft.Bot.Builder.Luis;
        ```

    1. Attribute of the WeatherDialog class:

        ```csharp
        [LuisModel("a5baea7f-7714-4c56-a13a-ea8b631d55ab", "a2afc4ddd6b540fe8e36d62cea3629f3")]
        ```
        Replace the first string with the modelId of your service and the second string with the subscription key of your service.

1. Additionally, we need to add the entity placeholders and the method routing for the newly created intent:

    1. Add the entity placeholder:

        ```csharp
        public const string Entity_Location = "locationName";
        ```

    1. Add the intent method

        ```csharp
        [LuisIntent("WeatherForLocation")]
        public async Task WeatherForLocation(IDialogContext context, LuisResult result)
        {
            EntityRecommendation location;
            if (result.TryFindEntity(Entity_Location, out location))
            {
                // Call to the OpenWeatherMap API
                await WeatherAPICaller.GetWeatherData(location.Entity, context.MakeMessage().Recipient.Id);
            }
            else
            {
                string m = $"Cannot get weather for unknown location. Try again with a query like: What is the weather in London";
                await context.PostAsync(m);
            }

            context.Wait(MessageReceived);
        }
        ```

1. The final **WeatherDialog** should now look like this:

    ```csharp
    using Microsoft.Bot.Builder.Dialogs;
    using Microsoft.Bot.Builder.Luis;
    using Microsoft.Bot.Builder.Luis.Models;
    using System;
    using System.Threading.Tasks;

    namespace WetterBot.Controllers
    {
        // LUIS Url: https://api.projectoxford.ai/luis/v2.0/apps/a5baea7f-7714-4c56-a13a-ea8b631d55ab?subscription-key=a2afc4ddd6b540fe8e36d62cea3629f3&verbose=true

        [Serializable]
        [LuisModel("a5baea7f-7714-4c56-a13a-ea8b631d55ab", "a2afc4ddd6b540fe8e36d62cea3629f3")]
        public class WeatherDialog : LuisDialog<object>
        {
            public const string Entity_Location = "locationName";

            [LuisIntent("")]
            public async Task None(IDialogContext context, LuisResult result)
            {
                string message = $"Sorry I did not understand: {result.Query}";
                await context.PostAsync(message);
                context.Wait(MessageReceived);
            }

            [LuisIntent("WeatherForLocation")]
            public async Task WeatherForLocation(IDialogContext context, LuisResult result)
            {
                EntityRecommendation location;
                if (result.TryFindEntity(Entity_Location, out location))
                {
                    // Call to the OpenWeatherMap API
                    await WeatherAPICaller.GetWeatherData(location.Entity, context.MakeMessage().Recipient.Id);
                }
                else
                {
                    string m = $"Cannot get weather for unknown location. Try again with a query like: What is the weather in London";
                    await context.PostAsync(m);
                }

                context.Wait(MessageReceived);
            }
        }
    }


    ```

    **Comment** This will not compile, because of the missing WeatherAPICaller class. We will create this in the next chapter.

## Connect Bot to Weather API ##

... 

## Further Steps ##

1. To learn about another way to program bots, try the [Form Flow Bot](https://github.com/Danielius1012/BotLabs/tree/master/Bot_Builder/2_Form_Flow_Bot) lab.

1. If you want to get to know a even more agile and Cloud-optimized way to develop a bot, check out the labs on the [Azure Bot Service](https://github.com/Danielius1012/BotLabs/tree/master/Azure_Bot_Service)

