import express from 'express';
import multer from 'multer';
import cors from 'cors';
import { GoogleGenAI } from '@google/genai';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

const upload = multer({ storage: multer.memoryStorage() });

// Initialize Gemini Client
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

app.post('/api/report', upload.single('image'), async (req, res) => {
  try {
    const lat = req.body.lat;
    const lng = req.body.lng;
    const file = req.file;

    if (!file) {
      return res.status(400).json({ error: 'No image uploaded' });
    }

    // Prepare image for Gemini Vision model
    const imagePart = {
      inlineData: {
        data: file.buffer.toString('base64'),
        mimeType: file.mimetype,
      },
    };

    const prompt = `Analyze this civic issue photo. Output JSON ONLY with this structure:
    {
      "category": "Pothole / Uncollected Garbage / Street Light Failure / Water Leakage / Other",
      "priority": "Critical / High / Medium / Low",
      "description": "Short professional 1-sentence description of the issue.",
      "department": "Name of responsible municipal department (e.g. Roads & Infrastructure, Waste Management, Public Works)",
      "estimatedResolutionTime": "e.g. 24 Hours, 48 Hours"
    }`;

    const response = await ai.models.generateContent({
      model: 'gemini-2.5-flash',
      contents: [prompt, imagePart],
      config: {
        responseMimeType: 'application/json'
      }
    });

    const aiOutput = JSON.parse(response.text);

    // Append geotag coordinates
    const ticketData = {
      ...aiOutput,
      lat: parseFloat(lat),
      lng: parseFloat(lng),
      id: `TICKET-${Math.floor(1000 + Math.random() * 9000)}`
    };

    console.log('Generated Ticket:', ticketData);
    res.json(ticketData);

  } catch (error) {
    console.error('AI Processing Error:', error);
    res.status(500).json({ error: 'Failed to process issue' });
  }
});

app.listen(3000, () => {
  console.log('CivicPulse MVP Server running on http://localhost:3000');
});