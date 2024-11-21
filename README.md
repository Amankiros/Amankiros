//+------------------------------------------------------------------+
//| ICT Strategy EA                                                  |
//+------------------------------------------------------------------+
input double RiskPercentage = 1.0;           // Risk per trade (%)
input int StopLossPips = 20;                 // Stop Loss in pips
input int TakeProfitPips = 40;               // Take Profit in pips
input int StartHourLondon = 8;               // London Session Start (GMT)
input int EndHourLondon = 17;                // London Session End (GMT)
input int StartHourNY = 13;                  // NY Session Start (GMT)
input int EndHourNY = 22;                    // NY Session End (GMT)

// Supported pairs
string Pairs[] = {"EURUSD", "GBPUSD", "USDJPY", "XAUUSD", "US30", "NAS100"};

// Variables for ICT logic
double liquidityHigh, liquidityLow;
double fvgStart, fvgEnd, orderBlockPrice;
bool marketStructureShift;

//+------------------------------------------------------------------+
//| OnInit Function: Initialization                                  |
//+------------------------------------------------------------------+
int OnInit() {
   Print("ICT EA Initialized");
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| OnTick Function: Main Logic                                      |
//+------------------------------------------------------------------+
void OnTick() {
   string symbol = Symbol();
   if (!ArrayContains(Pairs, symbol)) return;

   int hour = TimeHour(TimeCurrent());
   if (!IsWithinTradingSession(hour)) return;

   // Identify Liquidity Zones
   liquidityHigh = iHigh(NULL, PERIOD_H1, 1);
   liquidityLow = iLow(NULL, PERIOD_H1, 1);

   // Identify Fair Value Gap (FVG)
   fvgStart = iLow(NULL, PERIOD_H1, 2);
   fvgEnd = iHigh(NULL, PERIOD_H1, 3);

   // Detect Market Structure Shift (MSS)
   marketStructureShift = DetectMSS();

   // Trading logic
   if (marketStructureShift) {
      if (iClose(NULL, PERIOD_H1, 0) > fvgStart && iClose(NULL, PERIOD_H1, 0) < fvgEnd) {
         PlaceTrade(ORDER_BUY);
      } else if (iClose(NULL, PERIOD_H1, 0) < fvgStart && iClose(NULL, PERIOD_H1, 0) > fvgEnd) {
         PlaceTrade(ORDER_SELL);
      }
   }
}

//+------------------------------------------------------------------+
//| Helper Functions                                                 |
//+------------------------------------------------------------------+

bool DetectMSS() {
   double lastClose = iClose(NULL, PERIOD_H1, 1);
   double currentClose = iClose(NULL, PERIOD_H1, 0);
   return (currentClose > liquidityHigh || currentClose < liquidityLow);
}

bool IsWithinTradingSession(int hour) {
   return ((hour >= StartHourLondon && hour <= EndHourLondon) ||
           (hour >= StartHourNY && hour <= EndHourNY));
}

void PlaceTrade(int tradeType) {
   double lotSize = CalculateLotSize(RiskPercentage, StopLossPips);
   double price = (tradeType == ORDER_BUY) ? SymbolInfoDouble(Symbol(), SYMBOL_ASK) : SymbolInfoDouble(Symbol(), SYMBOL_BID);
   double sl = (tradeType == ORDER_BUY) ? price - StopLossPips * _Point : price + StopLossPips * _Point;
   double tp = (tradeType == ORDER_BUY) ? price + TakeProfitPips * _Point : price - TakeProfitPips * _Point;

   MqlTradeRequest request;
   MqlTradeResult result;

   request.action = TRADE_ACTION_DEAL;
   request.symbol = Symbol();
   request.volume = lotSize;
   request.type = tradeType;
   request.price = price;
   request.sl = sl;
   request.tp = tp;
   request.deviation = 3;
   request.magic = 123456;  // Magic number
   request.comment = "ICT EA Trade";

   if (OrderSend(request, result)) {
      Print("Trade placed successfully");
   } else {
      Print("Trade failed: ", result.retcode);
   }
}

double CalculateLotSize(double riskPercent, int stopLossPips) {
   double riskAmount = AccountInfoDouble(ACCOUNT_BALANCE) * (riskPercent / 100);
   return NormalizeDouble(riskAmount / (stopLossPips * _Point * SymbolInfoDouble(Symbol(), SYMBOL_TRADE_CONTRACT_SIZE)), 2);
}
