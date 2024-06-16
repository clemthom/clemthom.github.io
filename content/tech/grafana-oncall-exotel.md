---
title: "Grafana Oncall - Exotel call provider"
summary: Using exotel call provider with grafana oncall Open source version
date: 2024-06-14
tags: ["grafana","oncall","exotel"]
author: "Clement Thomas"
---

Grafana Oncall is an incident response tool similar to **Pagerduty**, **OpsGenie** etc. Grafana oncall has an Open source version which can be installed and setup locally and provides an option to use a custom phone provider of our choice, as of now they support **exotel, twilio and zvonok**. In this post, we will discuss on using exotel as call provider with Open source grafana oncall installation.

As mentioned in the [documentation](https://grafana.com/docs/oncall/latest/set-up/open-source/#supported-phone-providers) 

* we first modify the setting `GRAFANA_CLOUD_NOTIFICATIONS_ENABLED` to `False` to disable the Grafana OSS and cloud connector. 
* Next we set the `PHONE_PROVIDER` to `exotel`
* `EXOTEL_ACCOUNT_SID` can be found under `DEVELOPER SETTINGS->API Settings`. It will also be in the URL. ie `https://my.exotel.com/example/apisettings/... `. example is the SID in the given URL. 
* Under `DEVELOPER SETTINGS->API Settings` itself, you can create a new API key (username) and API Token password and set the same as `EXOTEL_API_KEY` and `EXOTEL_API_TOKEN`
* Exotel provides support for [applets](https://developer.exotel.com/applet) or apps that provides the ability to build custom workflows using exotel. For instance, We use **Greeting** applet to set a recorded voice to greet the customer in call, **Gather** to take numeric information from the users during the call etc. For now say lets create a Greeting applet, that will say `You have an alert`. In greeting applet, you can input the text and let the robot like voice read it out, upload a recorded audio file, or record one live or choose one from the library. More on it in the [documentation](https://developer.exotel.com/applet). Once you create the app, the exotel UI will show the app id which should be mentioned in `EXOTEL_APP_ID`. Since we are using a greeting applet app id, whenever there is an alert, exotel call to an oncall engineer will say `You have an alert`. 
* Some call providers support to specify the text to read out aloud in the call, which would be useful to read out the alert information to the oncall user. But Exotel applets as of now doesnt support it directly. 
* `EXOTEL_CALLER_ID` is the Exophone / Exotel virtual number, which can be found under `MANAGE->Exophones`.
* Whenever a grafana oncall user adds or updates his/her phone number, grafana oncall verifies the same.With exotel phone provider an SMS text message is triggered with a verification code which the user needs to input to verify the phone number. 
* You will need to specify `EXOTEL_SMS_DLT_ENTITY_ID` which is the Entity / Company ID registered on DLT (Distributed Ledger Technology) portal of operators in India. This parameter is optional in the Exotel SMS api for International numbers, but mandatory for Indian numbers. More information on how to register and obtain a DLT entity ID is documented in [exotel support documentation](https://support.exotel.com/support/solutions/articles/3000096504-trai-regulations-on-commercial-communications-dlt-portal-sms-in-india)
* With **DLT** and **TRAI** every SMS template needs to be registered and if the SMS sent doesn't match the specified template, the SMS doesnt get delivered. For now we will register a template `Your verification code is xxxxxx` where `xxxxxx` will be a 6 digit verification code sent to the user. We also specify a 5 character `sender id` when registering a template. Say if your company name is `Example.com`, the sender id can be something like `EXAMPL` or something similar. We also get a DLT Template ID for each registered template.
* Say if the `sender id` registered is `EXAMPL` we specify the same in `EXOTEL_SMS_SENDER_ID`
* We specify the verification template format in `EXOTEL_SMS_VERIFICATION_TEMPLATE` as `Your verification code is $verification_code`
* We specify the DLT entity id in `EXOTEL_SMS_DLT_ENTITY_ID`, Please note this is the DLT entity id and not the template id. 

Thats a lot of configuration, Now we should be all set to use exotel with grafana oncall. 
