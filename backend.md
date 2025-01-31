
const express = require("express");
const mongoose = require("mongoose");
const redis = require("redis");
const cors = require("cors");
const dotenv = require("dotenv");
const faqRoutes = require("./routes/faqRoutes");
const { GoogleTranslator } = require("@vitalets/google-translate-api");
const AdminBro = require("admin-bro");
const AdminBroExpress = require("@admin-bro/express");
const AdminBroMongoose = require("@admin-bro/mongoose");

dotenv.config();

const app = express();
const PORT = process.env.PORT || 5000;

// Middleware
app.use(express.json());
app.use(cors());

// Connect to MongoDB
mongoose
  .connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log("MongoDB connected"))
  .catch((err) => console.error("MongoDB connection error:", err));

// Redis client setup
const redisClient = redis.createClient({ url: process.env.REDIS_URL });
redisClient.connect();
redisClient.on("connect", () => console.log("Redis connected"));
redisClient.on("error", (err) => console.error("Redis error:", err));

// FAQ Model
const faqSchema = new mongoose.Schema({
  question: { type: String, required: true },
  answer: { type: String, required: true },
  translations: { type: Map, of: String },
});
const FAQ = mongoose.model("FAQ", faqSchema);

// Admin Panel
AdminBro.registerAdapter(AdminBroMongoose);
const adminBro = new AdminBro({
  databases: [mongoose],
  rootPath: "/admin",
});
const adminRouter = AdminBroExpress.buildRouter(adminBro);
app.use(adminBro.options.rootPath, adminRouter);

// Routes
app.use("/api/faqs", faqRoutes);

// Start Server
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

// backend/routes/faqRoutes.js
const express = require("express");
const router = express.Router();
const FAQ = require("../models/faqModel");
const redisClient = require("../index");
const translate = require("@vitalets/google-translate-api");

router.get("/", async (req, res) => {
  const lang = req.query.lang || "en";
  
  try {
    const cachedData = await redisClient.get(`faqs_${lang}`);
    if (cachedData) return res.json(JSON.parse(cachedData));

    const faqs = await FAQ.find();
    const translatedFAQs = await Promise.all(
      faqs.map(async (faq) => {
        if (faq.translations && faq.translations[lang]) {
          return { question: faq.translations[lang], answer: faq.translations[lang] };
        }
        const translatedQuestion = (await translate(faq.question, { to: lang })).text;
        const translatedAnswer = (await translate(faq.answer, { to: lang })).text;
        return { question: translatedQuestion, answer: translatedAnswer };
      })
    );
    
    await redisClient.setEx(`faqs_${lang}`, 3600, JSON.stringify(translatedFAQs));
    res.json(translatedFAQs);
  } catch (error) {
    res.status(500).json({ message: "Error fetching FAQs", error });
  }
});

module.exports = router;

// backend/models/faqModel.js
const mongoose = require("mongoose");

const faqSchema = new mongoose.Schema({
  question: { type: String, required: true },
  answer: { type: String, required: true },
  translations: { type: Map, of: String },
});

module.exports = mongoose.model("FAQ", faqSchema);

// Dockerfile
FROM node:16
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "index.js"]

// docker-compose.yml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - redis
      - mongo
    environment:
      MONGO_URI: "mongodb+srv://Sachin:Sachin4465@cluster0.snylryw.mongodb.net/faq"
      REDIS_URL: "redis://redis:6379"
  mongo:
    image: mongo
    ports:
      - "27017:27017"
  redis:
    image: redis
    ports:
      - "6379:6379"
