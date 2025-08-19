# Assignment-1
Cricbuzz Live Stats: Real-Time Cricket Insights &amp; SQL-Based Analytics

A complete starter blueprint to build a real‑time cricket analytics platform with Python, Streamlit, SQL, and REST/JSON APIs. This includes:

End‑to‑end architecture & folder structure

Production‑ready SQL schema (with indexes)

Data ingestion & API adapter layer

Streamlit multipage app (live, stats, SQL lab, CRUD, home)

25 fully‑formed SQL practice queries (beginner → advanced)

Security, testing, and deployment notes

1) High‑Level Architecture

Flow: API → Ingestion (Python) → Staging Tables → Transform → Core Schema → Analytics Views → Streamlit UI

┌──────────────────────────┐
│ Cricket REST API (JSON) │
└──────────────┬───────────┘
│ requests
▼
┌──────────────────┐
│ Ingestion Jobs │ (Python: /ingestion)
└───────┬──────────┘
│ upsert
▼
┌─────────────────────┐
│ SQL Database │ (MySQL/Postgres/SQLite)
│ • staging_* │ (raw API snapshots)
│ • core tables │ (normalized model)
│ • views/marts │ (analytics)
└─────────┬───────────┘
│ queries
▼
┌─────────────┐
│ Streamlit UI│ (pages: Home, Live, Top Stats, SQL Lab, CRUD)
└─────────────┘

2) Folder Structure (Modular)

3) cricbuzz_livestats/
├─ app/
│ ├─ main.py # Streamlit entry
│ ├─ pages/
│ │ ├─ 1_🏠_Home.py
│ │ ├─ 2_📺_Live_Matches.py
│ │ ├─ 3_📊_Top_Player_Stats.py
│ │ ├─ 4_🔍_SQL_Analytics_Lab.py
│ │ └─ 5_🛠️_CRUD_Operations.py
│ └─ components/
│ ├─ cards.py # small UI widgets
│ └─ charts.py # matplotlib helpers
├─ core/
│ ├─ schema.sql # DDL + indexes
│ ├─ seed.sql # minimal seed data
│ └─ views.sql # analytics views/CTEs
├─ ingestion/
│ ├─ fetch_api.py # HTTP layer
│ ├─ normalize.py # JSON→rows mapping
│ └─ upsert.py # DB upsert (MERGE/ON CONFLICT)
├─ services/
│ └─ api_adapter.py # wraps provider (Cricbuzz/others)
├─ utils/
│ ├─ db_connection.py # DB connectors (SQLite/MySQL/PG)
│ ├─ settings.py # env vars & config
│ └─ logging.py # structured logs
├─ tests/
│ ├─ test_schema.sql
│ └─ test_etl.py
├─ requirements.txt
├─ README.md
└─ .env.example

3) Database Schema (DDL)

Works on PostgreSQL / MySQL (minor tweaks). Use SERIAL/BIGSERIAL (PG) or AUTO_INCREMENT (MySQL). For SQLite, use INTEGER PRIMARY KEY.

3.1 Core Entities

-- Teams
);


-- Venues
CREATE TABLE venues (
venue_id BIGINT PRIMARY KEY,
venue_name VARCHAR(120) NOT NULL,
city VARCHAR(80),
country VARCHAR(50),
capacity INT
);


-- Series/Tournaments
CREATE TABLE series (
series_id BIGINT PRIMARY KEY,
series_name VARCHAR(160) NOT NULL,
match_type VARCHAR(20), -- Test, ODI, T20I, T20, etc.
host_country VARCHAR(50),
start_date DATE,
end_date DATE,
total_matches INT
);


-- Matches
CREATE TABLE matches (
match_id BIGINT PRIMARY KEY,
series_id BIGINT,
match_desc VARCHAR(200), -- e.g., "IND vs AUS, 2nd ODI"
match_type VARCHAR(20), -- Test, ODI, T20I
match_date TIMESTAMP,
venue_id BIGINT,
team1_id BIGINT NOT NULL,
team2_id BIGINT NOT NULL,
toss_winner_id BIGINT,
toss_decision VARCHAR(10), -- bat/bowl
match_winner_id BIGINT,
victory_margin VARCHAR(40), -- e.g., "23 runs", "5 wickets"
status VARCHAR(40), -- scheduled/live/completed
json_blob JSON, -- raw snapshot for traceability (PG: JSONB)
FOREIGN KEY (series_id) REFERENCES series(series_id),
FOREIGN KEY (venue_id) REFERENCES venues(venue_id),
FOREIGN KEY (team1_id) REFERENCES teams(team_id),
FOREIGN KEY (team2_id) REFERENCES teams(team_id),
FOREIGN KEY (toss_winner_id) REFERENCES teams(team_id),
FOREIGN KEY (match_winner_id) REFERENCES teams(team_id)
);


-- Innings
CREATE TABLE innings (
innings_id BIGINT PRIMARY KEY,
match_id BIGINT NOT NULL,
batting_team_id BIGINT NOT NULL,
innings_number INT NOT NULL, -- 1..2 (ODI/T20), 1..4 (Tests)
runs INT,
wickets INT,
overs DECIMAL(4,1),
run_rate DECIMAL(5,2),
FOREIGN KEY (match_id) REFERENCES matches(match_id),
FOREIGN KEY (batting_team_id) REFERENCES teams(team_id)
);

Participation (which players played in a match and role)
FOREIGN KEY (player_id) REFERENCES players(player_id),
FOREIGN KEY (team_id) REFERENCES teams(team_id)
);


-- Batting (per innings per player)
CREATE TABLE batting_scorecards (
id BIGINT PRIMARY KEY,
innings_id BIGINT,
player_id BIGINT,
batting_pos INT,
runs INT,
balls INT,
fours INT,
sixes INT,
strike_rate DECIMAL(6,2),
dismissal VARCHAR(100),
not_out BOOLEAN DEFAULT FALSE,
FOREIGN KEY (innings_id) REFERENCES innings(innings_id),
FOREIGN KEY (player_id) REFERENCES players(player_id)
);


-- Bowling (per innings per player)
CREATE TABLE bowling_scorecards (
id BIGINT PRIMARY KEY,
innings_id BIGINT,
player_id BIGINT,
overs DECIMAL(4,1),
maidens INT,
runs_conceded INT,
wickets INT,
economy DECIMAL(6,2),
FOREIGN KEY (innings_id) REFERENCES innings(innings_id),
FOREIGN KEY (player_id) REFERENCES players(player_id)
);


-- Fielding (per innings per player)
CREATE TABLE fielding_stats (
id BIGINT PRIMARY KEY,
innings_id BIGINT,
player_id BIGINT,
catches INT DEFAULT 0,
stumpings INT DEFAULT 0,
runouts INT DEFAULT 0,
FOREIGN KEY (innings_id) REFERENCES innings(innings_id),
FOREIGN KEY (player_id) REFERENCES players(player_id)
);


-- Optional granular: Ball-by-ball
CREATE TABLE deliveries (
delivery_id BIGINT PRIMARY KEY,
innings_id BIGINT,
over_no INT,
ball_in_over INT,
striker_id BIGINT,
non_striker_id BIGINT,
bowler_id BIGINT,
runs_batsman INT,
runs_extras INT,
wicket_type VARCHAR(30),
dismissal_player_id BIGINT,
FOREIGN KEY (innings_id) REFERENCES innings(innings_id)
);

3.2 Indexes (Performance)

3.3 Helpful Views

- Player aggregates by format (derive format from matches.match_type)
CREATE VIEW v_player_batting_agg AS
SELECT p.player_id, p.full_name, m.match_type,
COUNT(bs.id) AS inns,
SUM(bs.runs) AS runs,
AVG(NULLIF(bs.balls,0)) AS avg_balls,
AVG(bs.strike_rate) AS avg_sr
FROM batting_scorecards bs
JOIN innings i ON i.innings_id = bs.innings_id
JOIN matches m ON m.match_id = i.match_id
JOIN players p ON p.player_id = bs.player_id
GROUP BY p.player_id, p.full_name, m.match_type;


CREATE VIEW v_player_bowling_agg AS
SELECT p.player_id, p.full_name, m.match_type,
COUNT(bw.id) AS inns,
SUM(bw.wickets) AS wkts,
SUM(bw.runs_conceded) AS runs_conceded,
SUM(bw.overs) AS overs
FROM bowling_scorecards bw
JOIN innings i ON i.innings_id = bw.innings_id
JOIN matches m ON m.match_id = i.match_id
JOIN players p ON p.player_id = bw.player_id
GROUP BY p.player_id, p.full_name, m.match_type;

Ingestion & API Adapter (Python)
4.1 utils/db_connection.py

import os
import sqlalchemy as sa
from sqlalchemy.engine import Engine


DB_URL = os.getenv("DB_URL", "sqlite:///cricbuzz.db")


_engine: Engine | None = None


def get_engine() -> Engine:
global _engine
if _engine is None:
_engine = sa.create_engine(DB_URL, pool_pre_ping=True, future=True)
return _engine

services/api_adapter.py

import requests
from typing import Any, Dict


class CricketAPI:
def __init__(self, base_url: str, api_key: str | None = None):
self.base_url = base_url.rstrip('/')
self.api_key = api_key


def _headers(self):
h = {"Accept": "application/json"}
if self.api_key:
h["x-api-key"] = self.api_key
return h

def get_live_matches(self) -> Dict[str, Any]:
url = f"{self.base_url}/live-matches"
r = requests.get(url, headers=self._headers(), timeout=20)
r.raise_for_status()
return r.json()


def get_match_scorecard(self, match_id: str) -> Dict[str, Any]:
url = f"{self.base_url}/matches/{match_id}/scorecard"
r = requests.get(url, headers=self._headers(), timeout=20)
r.raise_for_status()
return r.json()


def get_top_players(self, category: str) -> Dict[str, Any]:
url = f"{self.base_url}/stats/top/{category}"
r = requests.get(url, headers=self._headers(), timeout=20)
r.raise_for_status()
return r.json()

Swap base_url and auth to your data provider (Cricbuzz-compatible JSON). Keep the adapter thin to switch providers easily.

4.3 ingestion/normalize.py

from typing import Dict, Any, List


def normalize_match(json_obj: Dict[str, Any]) -> Dict[str, Any]:
# Map provider fields → our columns (example; adjust to real payload)
return {
"match_id": int(json_obj["id"]),
"series_id": int(json_obj["seriesId"]),
"match_desc": json_obj.get("name"),
