const express = require("express");
const bodyParser = require("body-parser");
const cors = require("cors");
const stripe = require("stripe")("YOUR_STRIPE_SECRET_KEY"); // replace with real key
const paypal = require("paypal-rest-sdk");
const path = require("path");

const app = express();
app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.raw({ type: "application/json" })); // Stripe webhooks
app.use(express.static(path.join(__dirname, "public"))); // serve index.html

let totalRaised = 4500; // placeholder — store in a DB for real use

// -------- STRIPE WEBHOOK --------
app.post("/webhook/stripe", (req, res) => {
  let event;
  try {
    event = JSON.parse(req.body);
  } catch (err) {
    return res.status(400).send(`Webhook error: ${err.message}`);
  }

  if (event.type === "checkout.session.completed") {
    const amount = event.data.object.amount_total / 100; // Stripe sends in cents
    totalRaised += amount;
  }

  res.json({ received: true });
});

// -------- PAYPAL CONFIG --------
paypal.configure({
  mode: "sandbox", // change to "live" in production
  client_id: "YOUR_PAYPAL_CLIENT_ID",
  client_secret: "YOUR_PAYPAL_SECRET"
});

// -------- PAYPAL WEBHOOK --------
app.post("/webhook/paypal", (req, res) => {
  const event = req.body;

  if (event.event_type === "PAYMENT.SALE.COMPLETED") {
    const amount = parseFloat(event.resource.amount.total);
    totalRaised += amount;
  }

  res.sendStatus(200);
});

// -------- FRONTEND API --------
app.get("/api/donations", (req, res) => {
  res.json({ total: totalRaised });
});

// -------- FALLBACK (serve index.html) --------
app.get("*", (req, res) => {
  res.sendFile(path.join(__dirname, "public", "index.html"));
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`✅ Server running on http://localhost:${PORT}`));
