# Steeleye-assignment

import datetime as dt
from typing import List, Optional
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field

app = FastAPI()

# Mock database data
class TradeDB:
    trades = []

    @classmethod
    def search_trades(cls, search: str):
        return [trade for trade in cls.trades if search.lower() in trade.instrument_id.lower()
                or search.lower() in trade.instrument_name.lower()
                or search.lower() in trade.counterparty.lower()
                or search.lower() in trade.trader.lower()]

    @classmethod
    def filter_trades(cls, asset_class: Optional[str] = None,
                      end: Optional[dt.datetime] = None,
                      max_price: Optional[float] = None,
                      min_price: Optional[float] = None,
                      start: Optional[dt.datetime] = None,
                      trade_type: Optional[str] = None):
        filtered_trades = cls.trades

        if asset_class:
            filtered_trades = [trade for trade in filtered_trades if trade.asset_class == asset_class]

        if end:
            filtered_trades = [trade for trade in filtered_trades if trade.trade_date_time <= end]

        if max_price:
            filtered_trades = [trade for trade in filtered_trades if trade.trade_details.price <= max_price]

        if min_price:
            filtered_trades = [trade for trade in filtered_trades if trade.trade_details.price >= min_price]

        if start:
            filtered_trades = [trade for trade in filtered_trades if trade.trade_date_time >= start]

        if trade_type:
            filtered_trades = [trade for trade in filtered_trades if trade.trade_details.buySellIndicator == trade_type]

        return filtered_trades

# Mock database initialization
TradeDB.trades = [
    {
        "asset_class": "Equity",
        "counterparty": "Counterparty1",
        "instrument_id": "AAPL",
        "instrument_name": "Apple Inc.",
        "trade_date_time": dt.datetime.now(),
        "trade_details": {
            "buySellIndicator": "BUY",
            "price": 150.0,
            "quantity": 100
        },
        "trade_id": "1",
        "trader": "John Doe"
    },
    # Add more mock trades here...
]

# Pydantic models
class TradeDetails(BaseModel):
    buySellIndicator: str = Field(description="A value of BUY for buys, SELL for sells.")
    price: float = Field(description="The price of the Trade.")
    quantity: int = Field(description="The amount of units traded.")


class Trade(BaseModel):
    asset_class: Optional[str] = Field(alias="assetClass", default=None, description="The asset class of the instrument traded. E.g. Bond, Equity, FX...etc")
    counterparty: Optional[str] = Field(default=None, description="The counterparty the trade was executed with. May not always be available")
    instrument_id: str = Field(alias="instrumentId", description="The ISIN/ID of the instrument traded. E.g. TSLA, AAPL, AMZN...etc")
    instrument_name: str = Field(alias="instrumentName", description="The name of the instrument traded.")
    trade_date_time: dt.datetime = Field(alias="tradeDateTime", description="The date-time the Trade was executed")
    trade_details: TradeDetails = Field(alias="tradeDetails", description="The details of the trade, i.e. price, quantity")
    trade_id: str = Field(alias="tradeId", default=None, description="The unique ID of the trade")
    trader: str = Field(description="The name of the Trader")


# Routes
@app.get("/trades", response_model=List[Trade])
def get_trades(
    search: Optional[str] = Query(None, description="Search for trades by counterparty, instrument ID, instrument name, or trader"),
    asset_class: Optional[str] = Query(None, description="Filter trades by asset class"),
    end: Optional[dt.datetime] = Query(None, description="Filter trades with tradeDateTime before this date"),
    max_price: Optional[float] = Query(None, description="Filter trades with price less than or equal to this value"),
    min_price: Optional[float] = Query(None, description="Filter trades with price greater than or equal to this value"),
    start: Optional[dt.datetime] = Query(None, description="Filter trades with tradeDateTime after this date"),
    trade_type: Optional[str] = Query(None, description="Filter trades by buySellIndicator (BUY or SELL)")
):
    trades = TradeDB.trades

    if search:
        trades = TradeDB.search_trades(search)

    if asset_class or end or max_price or min_price or start or trade_type:
        trades = TradeDB.filter_trades(asset_class, end, max_price, min_price, start, trade_type)

    return trades

@app.get("/trades/{trade_id}", response_model=Trade)
def get_trade(trade_id: str):
    for trade in TradeDB.trades:
        if trade["trade_id"] == trade_id:
            return trade
    return {"error": "Trade not found"}

