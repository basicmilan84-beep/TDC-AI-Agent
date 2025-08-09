ai-messenger-bot/
│
├── .env
├── package.json
├── README.md
└── server.js
{
  "name": "ai-messenger-bot",
  "version": "1.0.0",
  "description": "AI agent koji odgovara na email, WhatsApp i Viber poruke sa emotivnim tonom",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "axios": "^1.5.0",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "openai": "^4.20.0",
    "twilio": "^4.12.0",
    "nodemailer": "^6.9.3"
  }
}# TDC-AI-Agent

PORT=3000

OPENAI_API_KEY=tvoj-openai-api-kljuc

TWILIO_ACCOUNT_SID=tvom-twilio-account-sid
TWILIO_AUTH_TOKEN=tvom-twilio-auth-token
TWILIO_WHATSAPP_NUMBER=whatsapp:+1234567890

VIBER_TOKEN=tvom-viber-bot-token

GMAIL_USER=tvom@gmail.com
GMAIL_PASS=tvom-aplikacioni-lozinka-ili-token

import express from "express";
import bodyParser from "body-parser";
import dotenv from "dotenv";
import axios from "axios";
import { Configuration, OpenAIApi } from "openai";
import nodemailer from "nodemailer";
import twilio from "twilio";

dotenv.config();

const app = express();
app.use(bodyParser.json());

const PORT = process.env.PORT || 3000;

const openai = new OpenAIApi(new Configuration({
  apiKey: process.env.OPENAI_API_KEY,
}));

// Nodemailer setup za Gmail (koristi aplikacionu lozinku)
const mailTransporter = nodemailer.createTransport({
  service: 'gmail',
  auth: {
    user: process.env.GMAIL_USER,
    pass: process.env.GMAIL_PASS,
  },
});

// Twilio client
const twilioClient = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

// Funkcija za generisanje AI odgovora sa emotivnim tonom
async function generateReply({ userMessage, tone = "empathetic and warm" }) {
  const prompt = `
You are a helpful assistant that replies warmly and with emotional intelligence.
User message: """${userMessage}"""
Respond concisely with a ${tone} tone. Keep it polite and clear.
Include a short friendly closing sentence.
`;

  const response = await openai.createChatCompletion({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "You are empathetic and concise." },
      { role: "user", content: prompt }
    ],
    max_tokens: 300,
    temperature: 0.7,
  });

  return response.data.choices[0].message.content.trim();
}

// Endpoint za primanje poruka sa svih kanala
app.post("/webhook/incoming", async (req, res) => {
  try {
    const { provider, payload } = req.body;

    let channel, from, text;

    if (provider === "twilio") {
      channel = "whatsapp";
      from = req.body.From;
      text = req.body.Body;
    } else if (provider === "viber") {
      channel = "viber";
      from = payload.sender.id;
      text = payload.message.text;
    } else if (provider === "gmail") {
      channel = "email";
      from = payload.from;
      text = payload.snippet || payload.text;
    } else {
      return res.status(400).send("Unknown provider");
    }

    const reply = await generateReply({ userMessage: text });

    if (channel === "whatsapp") {
      await twilioClient.messages.create({
        from: process.env.TWILIO_WHATSAPP_NUMBER,
        to: from,
        body: reply,
      });
    } else if (channel === "viber") {
      await axios.post("https://chatapi.viber.com/pa/send_message", {
        receiver: from,
        min_api_version: 1,
        sender: { name: "AI Agent" },
        type: "text",
        text: reply,
      }, { headers: { "X-Viber-Auth-Token": process.env.VIBER_TOKEN } });
    } else if (channel === "email") {
      await mailTransporter.sendMail({
        from: process.env.GMAIL_USER,
        to: from,
        subject: "Odgovor na vaš email",
        text: reply,
      });
    }

    res.status(200).send("OK");
  } catch (error) {
    console.error("Error in webhook:", error);
    res.status(500).send("Server error");
  }
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
