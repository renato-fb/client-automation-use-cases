# 🍁 Complete lead capture and bulk messaging automation for canada immigration company

I've created a complete lead automation for 3 events in the upcoming month that is responsible for handling the complete registration and follow-up process through email and WhatsApp that takes place in a Meta ad form and on the client's website. Here's how I've done it and what technologies I've used:

## ⚙️ 1. Setting up the lead capturing environments

### 🖥️ 1.1 Adding a form to their website for lead capture

They use WordPress, so I simply added an HTML widget through the Elementor UI containing what I would basically call a full web page (HTML, CSS, and JS) that has its main purpose to show a form with name, email, phone number, and event location inputs and fire a request to an n8n automation (will be shown later). I did this in order to avoid needlessly downloading any popup plugins since their website is very basic functionality-wise. On the main page, there are 3 sections, each responsible for every event that is to occur in different cities. Whenever you click one of them, this popup will be displayed already set to send a payload with the chosen city in mind. The user can also choose which event they would like to attend through a select dropdown regardless of which section they clicked.

- **This is the popup:**

![Alt text](https://github.com/renato-fb/client-automation-use-cases/blob/main/02_canada-immigration-leads/assets/screenshots/expo-form.jpg?raw=true)

### ⓕ 1.2 Configuring a Meta dev account to send a request containing lead information captured in an ad form to a webhook in n8n.

The title is pretty self-explanatory. This was also pretty straightforward, even though I had a bit of a difficult time understanding how Meta's API works. I found that they have very confusing documentation regarding lead capturing, at least form-wise. This could also be because this is the first time I performed an integration with Meta.

## ⭐ 2. Creating the actual automations for lead processing

In this part, I used n8n (responsible for the bulk of the automation), Chatwoot (for keeping track of the numbers the automation has sent messages to) in combination with Evolution API and a phone number to send messages, and Supabase with PostgreSQL DB. All of this is self-hosted on a VPS that I also set up.

### 2.1 The main workflow for lead processing

- **How it works (lifecycle):** \
  -> User activates webhook with form data \
  -> Data is normalized into a global format for the rest of the automation regardless of its source and format. \
  -> Supabase verifies if the user already exists in the database through a unique key (nome, email, telefone, id_evento <- If all of these match, the workflow is stopped) \
  -> A QR code is generated and saved into the database in base64 format through a public API (https://api.qrserver.com/v1/) with the query parameter of the user ID. This QR code is to be scanned by a staff member at the entrance on the days of the event only in order to determine who showed up since the entry is free. (The platform that will be used for the QR code scanning was also created by me as it will be showcased later) \
  -> A text message and one of 3 images are sent (each containing different city texts according to the event chosen, since all 3 are in different cities). The text message and image are sent to the lead as follows:

![Alt text](https://github.com/renato-fb/client-automation-use-cases/blob/main/02_canada-immigration-leads/assets/screenshots/main-automation.jpg?raw=true)

All of the requests are sent using Evolution API \
-> The chosen city is filtered in order to fetch the right image URL and email description to be appended to the email that is to be also sent to the user
\
-> Finally, the status of the messages sent is saved in the database in JSON format

📄 _Refer to DB Architecture section for reference if necessary [here](#db_architecture)_

**_All of the different nodes you can see in the image that I didn't refer to are still in development and will be responsible for future features that are still in the backlog_**

### 2.2 The workflow responsible for capturing leads from Meta forms

![Alt text](https://github.com/renato-fb/client-automation-use-cases/blob/main/02_canada-immigration-leads/assets/screenshots/captacao-fb-leads-expo.jpg?raw=true)

This is a relatively simple workflow. It receives the request in the webhook by Meta whenever an ad form is filled, and the data pertaining to that lead is sent to the main workflow to be fully registered and processed.

## 📲 3. QR Code Scanning Page

When the day of the event arrives, there will be a need for attendees to have their QR code (which was generated previously and sent to their WhatsApp) scanned. This page was created for the person managing entry to the event to be able to scan these QR codes. Here's the UI:

![Alt text](https://github.com/renato-fb/client-automation-use-cases/blob/main/02_canada-immigration-leads/assets/screenshots/qr-admin.png?raw=true)

As soon as the QR code is read, a request is made to an automation that sets the status of the lead in the database to "checked-in". The database also has a column that counts how many times the QR code was read to be able to track how many people have entered, even if the same QR code is scanned multiple times (e.g., when it's shared with friends or family who will be attending the same event). I will not show this automation since its rather simple.

## DB Architecture

- **`id`**: Unique identifier (UUID) for each registration record.
- **`nome`**: The attendee's full name.
- **`telefone`**: The attendee's phone number (used for WhatsApp dispatches).
- **`email`**: The attendee's email address (used for email dispatches).
- **`created_at`**: Timestamp of when the lead was registered in the database.
- **`id_evento`**: Identifier that links the attendee to a specific event.

---

### Dispatch Control Columns (`disp_*`)

These columns store a JSON object (or `null`) that records the status of communication dispatches (e.g., `{ "status_wpp": true, "status_email": true, ... }`).

- **`disp_inscricao`**: Status of the dispatch sent at the time of registration.
- **`disp_dia_pos_inscricao`**: Status of the dispatch sent 1 day after registration.
- **`disp_30_dias_antes`**: Status of the dispatch sent 30 days before the event.
- **`disp_1_semana_antes`**: Status of the dispatch sent 1 week before the event.
- **`disp_3_dias_antes`**: Status of the dispatch sent 3 days before the event.
- **`disp_1_dia_antes`**: Status of the dispatch sent 1 day before the event.
- **`disp_dia_evento`**: Status of the dispatch sent on the day of the event.

---

### Check-in and Origin Columns

- **`qr_base64`**: A base64 string representing the attendee's unique QR Code for check-in.
- **`status`**: The general status of the attendee in the event (e.g., 'confirmed', 'pending', 'checked-in').
- **`qtd_scan`**: A numerical counter of how many times the `qr_base64` has been scanned.
- **`origem`**: The lead's acquisition source (e.g., 'meta', 'wordpress', 'ci').

## 🛠️ Technologies Used

![n8n](https://img.shields.io/badge/n8n-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white) ![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white) ![Evolution API](https://img.shields.io/badge/Evolution_API-25D366?style=for-the-badge&logo=whatsapp&logoColor=white) ![Chatwoot](https://img.shields.io/badge/Chatwoot-FF6B6B?style=for-the-badge&logo=chatwoot&logoColor=white) ![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white) ![Meta API](https://img.shields.io/badge/Meta_API-1877F2?style=for-the-badge&logo=meta&logoColor=white) ![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white) ![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white) ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)

