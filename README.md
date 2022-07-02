# Zhang United Groups X Someone
---

### Fancy mission statement:
> To build an full automated trading bot (server based or VPS) based on Trading View's webhook alert for entry, close and stop lost. Futher enchancement by sending

### In a nutshell: 
> Flask app receiving alerts from TradingView and automatically place an order or send the chart to Discord where you can choose whether to take the trade or not through a bot.

```
Project Cost: MYR 20,000.00
Project Timeframe: 3 months
Project Lead: Maximenz
```
---

## How does it work? What is the plan?

****Step 1:** Trading View upgrade. To be able to use TradingView webhooks we will need to subscribe to the pro plan for approximately 12$ per month.

**Step 2:** Write ANY trading strategy on Pinescript. (https://www.tradingview.com/pine-script-docs/en/v5/Introduction.html) and test it immediately on Trading View to make sure it is profitable (duh). I would go for something like: if we are above/under the 200 period EMA, RSI is overbought/oversold and we have a bearish/bullish engulfing candle we will either short or long the position trying to come back to the 200 period exponential moving average. For ease of use the stop-losses and take-profits are placed using "random" values of ATR.

![image](https://user-images.githubusercontent.com/21031516/176994718-1e32aef1-cbef-4d8a-bb96-9c20f4820523.png)

As you can see we have multiples exit positions and some are even at breakeven, our integration with the FTX api is compatible for this style of trading, you can set the % you want to close for these intermediary TP in the ```tp_close``` variable. 

However, if you want to keep it to the bottom line you can just set **one entry** and **one fixed exit**. For that you will need to uncomment the ```strategy.exit()``` lines for both ```if goLong``` and ```if goShort``` and remove the two following lines:

```js
// Execute buy or sell order
if goLong and notInTrade
    size = (strategy.equity*risk) / abs(close-long_stop_level)
    strategy.entry("long", strategy.long, qty=size, alert_message="entry")
    // strategy.exit("exit", "long", stop=long_stop_level, limit=long_profit_level, alert_message="exit")
    inLongPosition := true // to remove
    notInTrade := false // to remove
```

And delete all the *multiple TP* category.

```js
// ----- MULTIPLE TP -----
// ...
```

**Step 3:** Write and build the Phython Flash App Bot

Aside from automation and manual interefered through discord. All alerts and action must matched with master password from the owner (God)

As we've said before we need to set a route that will receive **POST requests** and load the request data in a json format, so it will be : 

```python
from flask import request

@app.route("/tradingview-to-webhook-order", methods=['POST'])
    def tradingview_webhook():
        data = json.loads(request.data)
```
we will need to compare the stored password with the one we received. So beforehand let's fetch it.

```python
webhook_passphrase_heroku = os.environ.get('WEBHOOK_PASSPHRASE')
webhook_passphrase = webhook_passphrase_heroku if webhook_passphrase_heroku != None else config.WEBHOOK_PASSPHRASE
```

The main password is saved with the Heroku environment variables so we will try to get this one first, however if you are deploying the app locally it won't find it so you will need to write it in the [config.py](config.py) file, otherwise just set it to *None*.

When that's done we can verify that the payload actually contains a password before check if it is the right one and finally calling the ```order()``` function from [orderapi.py](orderapi.py).

```python
import logbot

if 'passphrase' not in data.keys():
    logbot.logs(">>> /!\ No passphrase entered", True)
    return {
        "success": False,
        "message": "no passphrase entered"
    }

if data['passphrase'] != webhook_passphrase:
    logbot.logs(">>> /!\ Invalid passphrase", True)
    return {
        "success": False,
        "message": "invalid passphrase"
    }

orders = order(data)
return orders
```
**Step 4:** Discord Logs & Manual Intefering

```python
import logbot

logbot.logs(">>> Order message : 'entry'")
# or
logbot.logs(">>> /!\ Invalid passphrase", True)
```

This ```logs()``` function from [logbot.py](whateverdamnname.py) act as a console **log printer** but also sends the logs to a discord channel you've specified so that you can easily at any time **track your bot's actions**.

![image](https://user-images.githubusercontent.com/21031516/176995097-5496259c-b471-45fa-9c77-bb33a0d8232d.png)

You can create another channel for **error logs** where you would set notifications to high priority so that you will be quickly aware if anything goes wrong.

![image](https://user-images.githubusercontent.com/21031516/176995115-a95cc073-81fc-4b1b-af36-531a9d708632.png)

To do that it's pretty simple, you'll have to send a json post request to the discord webhook *(Discord server settings / Integrations / Webhooks / New webhooks)* which must contain a username, a content and optionally an avatar. If an error has occured you just set the *error* variable to *true* and the message is also send to the error channel.

```python
import requests

DISCORD_ERR_URL = "https://discord.com/api/webhooks/xxxxx"
DISCORD_LOGS_URL = "https://discord.com/api/webhooks/xxxxx"
logs_format = {
	"username": "logs",
	"avatar_url": "https://imgr.search.brave.com/ZYUP1jUoQ5tr1bxChZtL_sJ0LHz7mDhlhkLHxWxhnPM/fit/680/680/no/1/aHR0cHM6Ly9jbGlw/Z3JvdW5kLmNvbS9p/bWFnZXMvZGlzY29y/ZC1ib3QtbG9nby00/LnBuZw",
	"content": ""
}

def logs(message, error=False):
    print(message)
    try:
        json_logs = logs_format
        json_logs['content'] = message
        requests.post(DISCORD_LOGS_URL, json=json_logs)
        if error:
            requests.post(DISCORD_ERR_URL, json=json_logs)
    except:
        pass
```

&nbsp;

**Step 5:** Upload, setup and winner winner chicken dinner lo duh
