# Run WattWise

1. Extract the ZIP.
2. Open the **wattwise** folder in VS Code. Make sure you can see `package.json` inside the folder.
3. Open a terminal in that folder. Install Node.js 22.12 or newer if needed.
4. Run:

```sh
npm ci
npm run setup
npm run dev
```

5. Open **http://localhost:3000** and click **Explore the demo**.

The demo contains sample data and a fictional tariff. You can add appliances, record usage, start/stop timers and explore reports immediately. Its data stays in your browser.

For **real Google accounts and database storage**, follow `README.md`: start PostgreSQL, run migrations and enter your Google OAuth credentials in `.env`. Configure an optional OpenAI API key for live AI lookup. Confirm your provider and tariff before using bill estimates.

You do not need to run the terminal as administrator. Keep it open while the development server is running. Press **Ctrl+C** to stop the server.

If npm reports that `package.json` is missing, your terminal is in the wrong folder. Change into the extracted `wattwise` folder and run the commands there.
