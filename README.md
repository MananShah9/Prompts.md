const express = require('express');
const axios = require('axios');
const NodeCache = require('node-cache');
const crypto = require('crypto'); // For generating cache keys

const app = express();
const port = 7001;

// Create a cache instance with a 1 hour TTL (3600 seconds)
const cache = new NodeCache({ stdTTL: 3600, checkperiod: 600 }); // Check for expired keys every 10 minutes

const DMD_API_URL = "https://qsctoreporting.uk.hsbc:7000/DMD/table";

// Middleware to parse JSON request bodies
app.use(express.json());

// Helper function to generate cache key
function generateCacheKey(requestBody) {
    const hash = crypto.createHash('md5').update(JSON.stringify(requestBody)).digest('hex');
    return `dmd_api_cache_${hash}`;
}

// Proxy endpoint
app.get('/dmd/table', async (req, res) => {
  try {
    const responseDateFormat = req.query.responseDateFormat;
    if(responseDateFormat !== 'keyValue') {
      return res.status(400).json({ error: 'responseDateFormat=keyValue is mandatory' });
    }
    const requestBody = req.body;

    const cacheKey = generateCacheKey(requestBody);
    const cachedData = cache.get(cacheKey);
    
    if(cachedData) {
      console.log("Cache hit");
      return res.json(cachedData);
    }
    
    console.log("Cache miss - fetching from DMD");

    const dmdResponse = await axios.get(DMD_API_URL, {
      params: { responseDateFormat: responseDateFormat },
      data: requestBody,
      headers: {
        'Content-Type': 'application/json'
      },
      httpsAgent: new (require('https').Agent)({ rejectUnauthorized: false }) // Disable SSL verification for self-signed certs, should not be used in prod

    });

      const responseData = dmdResponse.data;
      cache.set(cacheKey, responseData); // Store response in cache
      res.json(responseData);


  } catch (error) {
      console.error('Error fetching data from DMD API:', error);
      res.status(500).json({ error: 'Failed to fetch data from DMD API' });
    }
});


app.listen(port, () => {
  console.log(`Proxy server listening on port ${port}`);
});

