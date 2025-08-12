package.json
server.js
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const { createClient } = require('@supabase/supabase-js');
const twilio = require('twilio');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// Supabase setup
const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

// Twilio setup
const client = twilio(process.env.TWILIO_SID, process.env.TWILIO_AUTH);

// Store appointments in memory (simple for now)
let appointments = [];

// Add appointment
app.post('/add-appointment', (req, res) => {
    const { name, phone, dateTime } = req.body;
    if (!name || !phone || !dateTime) {
        return res.status(400).json({ error: 'Missing fields' });
    }
    appointments.push({ name, phone, dateTime });
    res.json({ message: 'Appointment added' });
});

// Send reminders
app.get('/send-reminders', async (req, res) => {
    const now = new Date();
    const upcoming = appointments.filter(a => {
        const apptTime = new Date(a.dateTime);
        const diff = apptTime - now;
        return diff > 0 && diff <= 24 * 60 * 60 * 1000; // within 24 hours
    });

    for (let appt of upcoming) {
        await client.messages.create({
            body: `Reminder: ${appt.name}, you have a barbershop appointment on ${appt.dateTime}`,
            from: process.env.TWILIO_NUMBER,
            to: appt.phone
        });
    }

    res.json({ message: 'Reminders sent' });
});

app.use(express.static('public'));

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
