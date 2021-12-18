# bot-x9

[const express = require("express");
const router = express.Router();
const mongoose = require("mongoose");
const ccxt = require("ccxt");
const tulind = require("tulind");

require("../models/Candles");
const candlesTick = mongoose.model("candles");

module.exports = () =>{
    const exchange = new ccxt.binance();
exchange.apiKey = process.env.API_KEY 
  //;
exchange.secret = process.env.SECRET_KEY 
  //;


  const superCandles = async () => {
    let timeFunction = async () => {
      try {
        if (exchange.has["fetchOHLCV"]) {
          let sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
          let symbol = "DENT/ETH";
          let candles = await exchange.fetchOHLCV(symbol, "1h");

          const pairUpdate = symbol;
          const velas = candles;
          //console.log(velas[499]);
          const updateCandles = await candlesTick.findOne({ pair: pairUpdate });
          //console.log(updateCandles)

          updateCandles.candleStick = velas;
          await updateCandles.save();
          console.log(velas[499]);

          
          //<<<<<<<<<<<<indicators>>>>>>>>>>>>//
          const open = updateCandles.candleStick.map((d) => d[1]);
          const high = updateCandles.candleStick.map((d) => d[2]);
          const low = updateCandles.candleStick.map((d) => d[3]);
          const close = updateCandles.candleStick.map((d) => d[4]);

          let BBANDS;
          let RSI;

          tulind.indicators.rsi.indicator([close], [14], (err, res) => {
            if (err) console.log(err);
            //console.log(res[0].slice(-1)[0])
            RSI = res[0].slice(-1)[0];
          });

          tulind.indicators.sma.indicator([close], [200], (err, res) => {
            if (err) console.log(err);
            //console.log(res[0].slice(-1)[0]);
          });

          tulind.indicators.ema.indicator([close], [21], (err, res) => {
            if (err) console.log(err);
            //console.log(res[0].slice(-1)[0])
            //RSI = res[0].slice(-1)[0];
            //console.log(open);
          });

          tulind.indicators.macd.indicator([close], [12, 20, 9], (err, res) => {
            if (err) console.log(err);
            //console.log(res.length)
          });

          tulind.indicators.psar.indicator(
            [high, low],
            [0.02, 0.2],
            (err, res) => {
              if (err) console.log(err);
              //console.log(res[0].slice(-1)[0])
            }
          );

          tulind.indicators.bbands.indicator([close], [21, 2.0], (err, res) => {
            if (err) console.log(err);
            //console.log(res[0].slice(-1)[0])
            // console.log(res[1].slice(-1)[0])
            //console.log(res[2].slice(-1)[0])
            BBANDS = res[0].slice(-1)[0];
          });

          //<<<<<<<<<<<<<Create Order>>>>>>>>>>>>>>\\
          let alocation = 0.007;
          let quantity = alocation / close[499];
          let since = exchange.parse8601("2018-01-01T00:00:00Z");
          let limit = 5;
          //let symbol = "ETH/DENT";
          const amount = quantity; // DENT
          const price = close[499]; // ETH
          let buyOrder;
          let openOrders = await exchange.fetchOpenOrders(symbol, since, limit);
          let balance = await exchange.fetchBalance();

          console.log('Tem ordens abertas',!openOrders);
          console.log('Fechamento do candle' ,close[499]);
          console.log('RSI', RSI);
          console.log('BBANDS', BBANDS);
          console.log('Quantidade a comprar', amount);
          console.log('Preço de venda', price + price * 0.035);

          
          if (!openOrders && close[499] <= BBANDS && RSI <= 30) {
            buyOrder = await exchange.createLimitBuyOrder(
              symbol,
              amount,
              price
            );
            console.log(buyOrder);
          }

          if (balance.DENT.free >= amount && !openOrders) {
            price = price + price * 0.035;
            await exchange.createLimitSellOrder(symbol, amount, price);
          }

          ////////////////
        } else {
          console.log("não");
        }
      } catch (error) {
        console.log(error);
      }
    };

    setInterval(timeFunction, 60000);
  };

  superCandles();

}](url)


####dependecies
{
  "name": "ccxt-bot",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "dev": "nodemon index.js",
    "start": "index.js"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "axios": "^0.21.1",
    "body-parser": "^1.19.0",
    "ccxt": "^1.47.12",
    "cors": "^2.8.5",
    "express": "^4.17.1",
    "moment": "^2.29.1",
    "mongoose": "^5.12.3",
    "nodemon": "^2.0.7",
    "tulind": "^0.8.18"
  },
  "engines": {
    "node": "14.15.5",
    "npm": "6.14.11"
  }
}

####
const express = require("express");
const mongoose = require("mongoose");
const cors = require('cors')
const bodyParser = require('body-parser')

mongoose.connect(
  process.env.MONGODB_URL || "mongodb://localhost/trading",
  {
    useNewUrlParser: true,
    useCreateIndex: true,
    useUnifiedTopology: true,

  }
).then(item => {
  console.log('conectado com o banco')
}).catch((err)=>{
  console.log(err)
});

/////////////////Function///////////////////
const ccxt = require("ccxt");
const tulind = require("tulind");

require("./models/Candles");
const candlesTick = mongoose.model("candles");
require('./utils/trading');

//////////////////////////////////////////////

const app = express();
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({extended: false}))

const candlesRoute = require("./routes/candles");
const home = require("./routes/home");

const trading = require("./utils/trading");

app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", 'GET,PUT,POST,DELETE');

  app.use(cors());
  next();

})

setInterval(trading, 60000)



app.use("/", home);
app.use("/candles", candlesRoute);


const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log("Servidor Conectado");
});
