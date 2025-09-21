# WhatsApp Integration Documentation

This document provides a detailed guide to understanding and configuring the WhatsApp integration for Ava.

## WhatsApp Data Flow

The following diagram illustrates the end-to-end data flow for an interaction with Ava via WhatsApp.

```mermaid
sequenceDiagram
    participant User
    participant WhatsApp
    participant FastAPI_Backend as FastAPI Backend
    participant LangGraph_Agent as LangGraph Agent

    User->>+WhatsApp: Sends a message (text, audio, or image)
    WhatsApp->>+FastAPI_Backend: Forwards message to Webhook (/whatsapp_response)

    alt Webhook Verification (GET)
        FastAPI_Backend->>WhatsApp: Responds with challenge code
    else Message Processing (POST)
        FastAPI_Backend->>FastAPI_Backend: Parses incoming message
        alt Message Type
            case Text
                FastAPI_Backend->>FastAPI_Backend: Extracts text content
            case Audio
                FastAPI_Backend->>WhatsApp: Downloads audio file
                WhatsApp-->>FastAPI_Backend: Returns audio data
                FastAPI_Backend->>FastAPI_Backend: Transcribes audio to text
            case Image
                FastAPI_Backend->>WhatsApp: Downloads image file
                WhatsApp-->>FastAPI_Backend: Returns image data
                FastAPI_Backend->>FastAPI_Backend: Analyzes image content
        end
        FastAPI_Backend->>+LangGraph_Agent: Invokes agent with processed content
        LangGraph_Agent-->>-FastAPI_Backend: Returns final state (with response)

        FastAPI_Backend->>FastAPI_Backend: Determines response type (text, audio, or image)
        alt Response Type
            case Text
                FastAPI_Backend->>+WhatsApp: Sends text message
            case Audio
                FastAPI_Backend->>+WhatsApp: Uploads audio file to get media ID
                WhatsApp-->>-FastAPI_Backend: Returns media ID
                FastAPI_Backend->>+WhatsApp: Sends audio message with media ID
            case Image
                FastAPI_Backend->>+WhatsApp: Uploads image file to get media ID
                WhatsApp-->>-FastAPI_Backend: Returns media ID
                FastAPI_Backend->>+WhatsApp: Sends image message with media ID and caption
        end
        WhatsApp-->>-User: Delivers response message
    end
```

## Webhook Code Documentation

This section provides a detailed breakdown of the webhook's source code, located in `src/ai_companion/interfaces/whatsapp/whatsapp_response.py`.

### Overview

The webhook is a FastAPI endpoint that serves two primary purposes:
1.  **Webhook Verification**: It handles the `GET` request from Meta to verify the webhook's authenticity.
2.  **Message Processing**: It handles `POST` requests containing incoming messages from users and sends them to the LangGraph agent for processing.

### `whatsapp_handler` Function

This is the main function that handles all incoming requests to the `/whatsapp_response` endpoint.

- **`GET` Requests**: For `GET` requests, the function checks if the `hub.verify_token` in the query parameters matches the `WHATSAPP_VERIFY_TOKEN` environment variable. If they match, it returns the `hub.challenge` to complete the verification process.
- **`POST` Requests**: For `POST` requests, the function parses the JSON payload to extract the message details.

### Message Handling

The webhook can handle three types of messages: text, audio, and image.

- **Text Messages**: The text content is extracted directly from the message payload.
- **Audio Messages**: The `process_audio_message` helper function is called to download the audio file from WhatsApp's servers and transcribe it to text using the `SpeechToText` module.
- **Image Messages**: The `download_media` helper function is called to download the image. The `ImageToText` module is then used to analyze the image and generate a description. Any caption included with the image is also extracted.

### Agent Invocation and Response

Once the message content is processed, it is passed to the LangGraph agent using `graph.ainvoke`. The `from_number` is used as the `thread_id` to maintain conversation state.

After the agent has finished processing, the final state is retrieved to determine the response type (workflow) and the content of the response.

### Helper Functions

- **`download_media(media_id)`**: Downloads a media file (image or audio) from WhatsApp's servers using its media ID.
- **`process_audio_message(message)`**: Orchestrates the downloading and transcription of an audio message.
- **`send_response(...)`**: Sends a response back to the user. It can handle text, audio, and image responses. For media responses, it first calls `upload_media`.
- **`upload_media(media_content, mime_type)`**: Uploads a media file to WhatsApp's servers to get a `media_id` that can be used to send the media to the user.

## API Setup Instructions

This guide explains how to set up the WhatsApp Cloud API and obtain the necessary credentials to connect Ava to WhatsApp.

### Prerequisites

- A [Meta Developer Account](https://developers.facebook.com/docs/development/register).
- A [Meta Business App](https://developers.facebook.com/docs/development/create-an-app/). If you don't see an option to create a business app, select "Other" > "Next" > "Business" during app creation.

### Step 1: Set Up the WhatsApp Product

1.  From the "My Apps" screen in your Meta Developer account, select your Business App.
2.  If you have not already done so, add the "WhatsApp" product to your app. You will be prompted to "Set up" WhatsApp.
3.  This process will create a test WhatsApp Business Account (WABA), a test phone number, and a pre-approved "hello world" message template.

### Step 2: Obtain Your Credentials

Navigate to the **WhatsApp > API Setup** panel in your App Dashboard. Here you will find your credentials:

1.  **`WHATSAPP_TOKEN`**: Click the **Generate access token** button. This will generate a temporary access token that is valid for 24 hours. For production use, you should set up a permanent [System User Token](https://developers.facebook.com/docs/whatsapp/business-management-api/get-started#system-user-access-tokens).
2.  **`WHATSAPP_PHONE_NUMBER_ID`**: This is the ID of the test phone number that was created for you. You can find it under the "From" field in the "Send and receive messages" section.
3.  **`WHATSAPP_VERIFY_TOKEN`**: This is a token that you create yourself. It's used to verify that the webhook requests are coming from Meta. You can enter any string you like in the webhook configuration (see next step), and then set the same value for the `WHATSAPP_VERIFY_TOKEN` environment variable.

### Step 3: Configure the Webhook

1.  In the **WhatsApp > API Setup** panel, find the "Webhooks" section and click **Configure webhook**.
2.  In the "Callback URL" field, you will need to enter the public URL of your running Ava application, followed by `/whatsapp_response`. If you are running the application locally, you will need to use a tool like [ngrok](https://ngrok.com/) to expose your local server to the internet.
3.  In the "Verify token" field, enter the string you created for your `WHATSAPP_VERIFY_TOKEN`.
4.  Click **Verify and save**.

### Step 4: Subscribe to Webhook Events

1.  After configuring the webhook, you need to subscribe to the events you want to receive notifications for.
2.  In the "Webhooks" section, click **Manage**.
3.  Under "WhatsApp Business Account", find "messages" and click **Subscribe**.

Once you have completed these steps, you should have all the necessary credentials and your webhook will be configured to receive messages from WhatsApp.
