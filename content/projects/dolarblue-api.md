---
id: dolarblue-api
title: DolarBlue Rest API
description: Rest api for Argentinian Peso - 'Blue'
author: Gaston Otero
date: "2021-06-27"
---

## Introduction

Dolar Blue Rest API is a backend service for fetching the price of the "Dolar Blue" in Argentina from the most popular sources.

The "Dolar Blue" refers to how we call the unregulated US dollar in the parallel markets.

The Argentinian currency system is not a free market like other countries, here the argentinian peso can only be exchanged
in the Central Bank with many limitations, the average argentinian is unable to do it.

Because of this the average Argentinian has to resort to the "unregulated" currency market, but of course, that has its flaws: there is no standardized/centralized way to know the price of the US dollar. The parallel (BLUE) market might seem like a libertarian's utopia, everything is handled by pure supply and demand, but there is a catch: there are too many different sources to get the current price of the "Dolar Blue". This API serves as a way to at least try to centralize and group the most popular sources into only one place.

This project is not in favour or against the parallel currency market or the measures our govt. takes to handle the economy,
it is irrelevant to this program's scope. To this date, the python interpreter is agnostic about tribal human politics, although this may change
in the future years with a better development of artificial intelligence technologies.

The information is there, it is public and this program only tries to gather and organize it.

## Services

This API has two layers of communication: a REST API and a Telegram bot. It will also have a python SDK in the future for easier usage.

NOTE: All times are in UTC

### Rest API

The REST api has two main routes:

#### Getting the average dolarblue values

If you just want to get an updated buy and sell price use this route.
Each update cycle it takes the buy and sell values of each source that could be fetched and updates
the average of those sources in the cache.

Request type: GET

Example response:

`{ "buy_price": 200.7, "sell_price": 204.1, "average_price": 202.4, "date_time": "12-24-2021 17:14:06" }`

#### Fetching the sources

If you want to verify the sources use this URL, it returns a json object with each source and it's associated
dolarblue price values. The source's date represent where the source was last fetched. Sources are fetched
in intervals together, so if a source has an old date it means it failed many times. Failed sources are not
taken into account in the average dolarblue price, so they do not affect it.

Request type: GET

Example response:

`{ "infodolar": { "buy_price": 201, "sell_price": 204, "average_price": 202.5, "date_time": "12-24-2021 17:14:06" }, "lanacion": { "buy_price": 200, "sell_price": 204, "average_price": 202, "date_time": "12-24-2021 17:14:06" } }`

## Tech stack

This project uses Python as the main programming language.
It scrapes the different sources with beautifulsoup4 and saves the data to a Redis cache for quick access.
FastApi is used as the backend server for the REST API, and the python-telegram-bot package as the API layer for Telegram.

## Source code

This project is completely free, not only free as in "free beer", but also as in "free speech".

The complete source code can be found found in this [repository](https://github.com/gastonoterom/dolarblue).
