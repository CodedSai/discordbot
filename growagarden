import json
import math
import os
import time
from datetime import datetime
from pathlib import Path

import discord
from discord.ext import commands, tasks

# --- CONFIGURATION ---
TOKEN = os.getenv("DISCORD_TOKEN") or os.getenv("TOKEN") or "MTUxNjQ4ODc3Mjg3Mzc1MjY4Ng.GSEFc9.-RrGvRe3owghYpVFoNYNuH9QMTF6RhlHUr1Whs"
SEEDS_CHANNEL_ID = int(os.getenv("SEEDS_CHANNEL_ID", "1516490006879797368"))
GEARS_CHANNEL_ID = int(os.getenv("GEARS_CHANNEL_ID", "1516489563692990504"))
WEATHER_CHANNEL_ID = int(os.getenv("WEATHER_CHANNEL_ID", "1516468855419965603"))
AUTO_POST_INTERVAL_SECONDS = int(os.getenv("AUTO_POST_INTERVAL_SECONDS", "300"))

DATA_FILE = Path(__file__).resolve().parent / "index.html"
STATE_FILE = Path(__file__).resolve().parent / "bot_state.json"
STATE = {"seeds": None, "gears": None, "weather": None}

intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)

SUN_ICON = "https://cdn.discordapp.com/emojis/1515099760724541692.webp?size=96"
GEAR_ICON = "https://cdn.discordapp.com/emojis/1515138112991531080.webp?size=96"
WEATHER_ICONS = {
    "Day": "https://tr.rbxcdn.com/180DAY-a323f20a7993add8bc2bfbcdda899b3d/150/150/Image/Png/noFilter",
    "Sunset": "https://tr.rbxcdn.com/180DAY-54db2df8be59b9c761f7b8a73b33fadc/150/150/Image/Png/noFilter",
    "Moon": "https://tr.rbxcdn.com/180DAY-26e332227f0503988607808381cd10a0/150/150/Image/Png/noFilter",
    "Goldmoon": "https://tr.rbxcdn.com/180DAY-d6887079cbf7b7d55f75e94aed7046a2/150/150/Image/Png/noFilter",
    "Rainbow Moon": "https://tr.rbxcdn.com/180DAY-72567c570d6d5f95aa4c12c0cc05780f/150/150/Image/Png/noFilter",
    "Bloodmoon": "https://tr.rbxcdn.com/180DAY-99bdbd298ac307ea4d2a9f39481507a5/150/150/Image/Png/noFilter",
}

RANK = {
    "Common": 1,
    "Uncommon": 2,
    "Rare": 3,
    "Epic": 4,
    "Legendary": 5,
    "Mythic": 6,
    "Super": 7,
}

WEATHER_SKIP = {"Day", "Sunset", "Moon"}
DATA = None


def now():
    return int(time.time())


def load_data():
    if not DATA_FILE.exists():
        raise FileNotFoundError(f"Could not find {DATA_FILE}")

    text = DATA_FILE.read_text(encoding="utf-8")
    start = text.index("let DATA =")
    start = text.index("{", start)

    depth = 0
    end = None
    for idx, ch in enumerate(text[start:], start):
        if ch == "{":
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0:
                end = idx + 1
                break

    if end is None:
        raise ValueError("Could not parse DATA from index.html")

    json_text = text[start:end]
    return json.loads(json_text)


def load_state():
    global STATE
    if STATE_FILE.exists():
        try:
            raw = STATE_FILE.read_text(encoding="utf-8")
            STATE = json.loads(raw)
        except Exception:
            STATE = {"seeds": None, "gears": None, "weather": None}
    else:
        STATE = {"seeds": None, "gears": None, "weather": None}


def save_state():
    try:
        STATE_FILE.write_text(json.dumps(STATE), encoding="utf-8")
    except Exception:
        pass


def format_price(value):
    if value >= 1_000_000_000:
        return f"{round(value / 1_000_000_000, 1)}B"
    if value >= 1_000_000:
        return f"{round(value / 1_000_000, 1)}M"
    if value >= 1_000:
        return f"{round(value / 1_000, 1)}K"
    return str(value)


def format_timestamp(ts, style="R"):
    return f"<t:{int(ts)}:{style}>"


def short_duration(seconds):
    seconds = max(0, int(seconds))
    if seconds < 3600:
        return f"{max(1, round(seconds / 60))}m"
    if seconds < 86400:
        hours = seconds // 3600
        minutes = round((seconds % 3600) / 60)
        return f"{hours}h {minutes}m" if minutes else f"{hours}h"
    days = seconds // 86400
    hours = round((seconds % 86400) / 3600)
    return f"{days}d {hours}h" if hours else f"{days}d"


def anchor_for(tab, data):
    return data.get("gearAnchor" if tab == "gears" else "seedAnchor", 0)


def evaluate_item(item, tab, data):
    period = data.get("period", 300)
    anchor = anchor_for(tab, data)
    q_list = item.get("q", [])
    count = data.get("count", len(q_list))
    index = math.floor((now() - anchor) / period)
    current = None
    next_restock = None

    if 0 <= index < count and q_list[index] > 0:
        current = {"t": anchor + index * period, "q": q_list[index]}

    for idx in range(max(index + 1, 0), count):
        if q_list[idx] > 0:
            next_restock = {"t": anchor + idx * period, "q": q_list[idx]}
            break

    return current, next_restock


def build_shop_embed(tab, data):
    shop_name = "Seed Shop" if tab == "seeds" else "Gear Shop"
    color = 0x74C043 if tab == "seeds" else 0x3A8FD6
    anchor = anchor_for(tab, data)
    period = data.get("period", 300)
    t = now()
    next_global = anchor if t < anchor else anchor + period * math.ceil((t - anchor) / period)
    next_time = format_timestamp(next_global, "R")
    short_time = datetime.fromtimestamp(next_global).strftime("%I:%M %p").lstrip("0")
    short_time_ts = format_timestamp(next_global, "t")
    duration_text = short_duration(next_global - t)

    items = data.get(tab, [])
    if tab == "gears":
        allowed = {
            "Basic Sprinkler",
            "Uncommon Sprinkler",
            "Rare Sprinkler",
            "Legendary Sprinkler",
            "Super Sprinkler",
            "Trowel",
            "Common Watering Can",
            "Super Watering Can",
            "Basic Pot",
        }
        items = [item for item in items if item.get("name") in allowed or "mushroom" in item.get("name", "").lower()]

    current_rows = []
    next_rows = []
    notable = []
    for item in items:
        current, next_restock = evaluate_item(item, tab, data)
        if current:
            current_rows.append((item, current))
        if next_restock:
            next_rows.append((item, next_restock))
            if RANK.get(item.get("rarity"), 0) >= RANK["Epic"]:
                notable.append((item, next_restock))

    current_rows.sort(key=lambda pair: (RANK.get(pair[0].get("rarity"), 99), pair[0].get("price", 0), pair[0].get("name", "")))
    next_rows.sort(key=lambda pair: (pair[1]["t"], RANK.get(pair[0].get("rarity"), 99), pair[0].get("price", 0), pair[0].get("name", "")))
    notable.sort(key=lambda pair: (pair[1]["t"], RANK.get(pair[0].get("rarity"), 99), pair[0].get("name", "")))

    return current_rows, next_rows, notable, shop_name, color, next_time, short_time, short_time_ts, duration_text, tab


def build_instock_embed(tab, data):
    current_rows, _, _, shop_name, color, next_time, short_time, short_time_ts, _, _ = build_shop_embed(tab, data)
    
    embed = discord.Embed(
        title="Grow a Garden — Current Stock",
        description=f"Shop Restocks in: {next_time} · {short_time_ts}",
        color=color,
    )
    embed.set_thumbnail(url=SUN_ICON if tab == "seeds" else GEAR_ICON)

    if current_rows:
        lines = [f"**{item.get('name', 'Unknown')}{' Seed' if tab == 'seeds' else ''}** x{current['q']} · {item.get('rarity', 'Unknown')}" for item, current in current_rows[:12]]
        embed.add_field(name="", value="\n".join(lines), inline=False)
    else:
        embed.description += " · No items currently in stock"

    return embed


def build_next_restock_embed(tab, data):
    _, next_rows, _, shop_name, color, next_time, short_time, short_time_ts, _, _ = build_shop_embed(tab, data)
    
    embed = discord.Embed(
        title="Grow a Garden — Predicted Restock",
        description=f"Next Restock {next_time} · {short_time_ts}",
        color=color,
    )

    if next_rows:
        lines = [f"**{item.get('name', 'Unknown')}{' Seed' if tab == 'seeds' else ''}** x{next_restock['q']} · {item.get('rarity', 'Unknown')} · {format_timestamp(next_restock['t'], 'R')}" for item, next_restock in next_rows[:12]]
        embed.add_field(name="", value="\n".join(lines), inline=False)
    else:
        embed.description += " · No upcoming restocks"

    return embed


def build_notable_embed(tab, data):
    _, _, notable, shop_name, color, _, short_time, short_time_ts, _, _ = build_shop_embed(tab, data)
    
    embed = discord.Embed(
        title="GrowAGarden 2 — Upcoming Predicted Stocks",
        color=color,
    )
    embed.set_footer(text=f"Saii OnTop · updates every minute • {short_time}")

    if notable:
        lines = [f"**{item.get('name', 'Unknown')}{' Seed' if tab == 'seeds' else ''}** - {item.get('rarity', 'Unknown')} - {format_timestamp(next_restock['t'], 'R')}" for item, next_restock in notable[:10]]
        embed.add_field(name="Notable Upcoming", value="\n".join(lines), inline=False)
    else:
        embed.description += " · No notable upcoming items"

    return embed


def weather_at(timestamp, data):
    weather = data.get("weather", {})
    clen = weather.get("clen", 0)
    if clen <= 0:
        return None

    cycle = timestamp // clen
    into = timestamp - cycle * clen
    phase_index = 0
    for idx, phase in enumerate(weather.get("phases", [])):
        if into >= phase["offset"]:
            phase_index = idx

    seq_index = cycle - weather.get("startCycle", 0)
    name = None
    if 0 <= seq_index < len(weather.get("seq", [])):
        seq_row = weather["seq"][seq_index]
        if phase_index < len(seq_row):
            name = seq_row[phase_index]

    next_offset = weather["phases"][phase_index + 1]["offset"] if phase_index + 1 < len(weather.get("phases", [])) else clen
    return {
        "name": name,
        "phase_name": weather["phases"][phase_index]["name"],
        "secs_left": cycle * clen + next_offset - timestamp,
        "cycle": cycle,
        "phase_index": phase_index,
    }


def upcoming_weather(data, limit=10):
    current = weather_at(now(), data)
    if not current or not current.get("name"):
        return []

    weather = data.get("weather", {})
    entries = []
    cycle = current["cycle"]
    phase_index = current["phase_index"]
    clen = weather.get("clen", 0)
    timestamp = now()

    attempts = 0
    while len(entries) < limit and attempts < limit * 5:
        phase_index += 1
        if phase_index >= len(weather.get("phases", [])):
            phase_index = 0
            cycle += 1

        seq_index = cycle - weather.get("startCycle", 0)
        if seq_index < 0 or seq_index >= len(weather.get("seq", [])):
            break

        seq_row = weather["seq"][seq_index]
        name = seq_row[phase_index] if phase_index < len(seq_row) else None
        if name and name not in WEATHER_SKIP:
            start_time = cycle * clen + weather["phases"][phase_index]["offset"]
            entries.append({"name": name, "secs": start_time - timestamp})
        attempts += 1

    return entries


def build_weather_embed(data):
    embed = discord.Embed(title="Grow a Garden Weather", description="Predicted weather schedule from Grow a Garden.", color=0x3B2C6E)
    current = weather_at(now(), data)
    if current and current.get("name") and current["name"] not in WEATHER_SKIP:
        embed.set_thumbnail(url=WEATHER_ICONS.get(current["name"], SUN_ICON))
        embed.add_field(
            name=f"Now: {current['name']}",
            value=(
                f"Phase: {current['phase_name']}\n"
                f"Ends in: {short_duration(current['secs_left'])}"
            ),
            inline=False,
        )
    else:
        embed.set_thumbnail(url=SUN_ICON)
        embed.add_field(name="Now", value="No special weather event is active right now.", inline=False)

    upcoming = upcoming_weather(data, limit=10)
    if not upcoming:
        embed.add_field(name="Upcoming", value="No special weather events found.", inline=False)
        embed.set_footer(text="Weather may vary per server")
        return embed

    for entry in upcoming:
        timestamp = now() + entry['secs']
        embed.add_field(name=entry["name"], value=f"{format_timestamp(timestamp, 'R')} · {format_timestamp(timestamp, 't')}", inline=False)

    embed.set_footer(text="Weather may vary per server")
    return embed


async def get_channel(channel_id):
    if channel_id <= 0:
        return None
    channel = bot.get_channel(channel_id)
    if channel is None:
        try:
            channel = await bot.fetch_channel(channel_id)
        except Exception:
            return None
    return channel


async def upsert_embed(channel_id, state_key, embed, content=""):
    channel = await get_channel(channel_id)
    if channel is None:
        raise ValueError(f"Channel {channel_id} is not accessible or not configured.")

    message = None
    message_id = STATE.get(state_key)
    if message_id:
        try:
            message = await channel.fetch_message(int(message_id))
        except (discord.NotFound, discord.Forbidden, discord.HTTPException):
            message = None

    if message:
        await message.edit(content=content, embed=embed)
    else:
        message = await channel.send(content=content, embed=embed)
        STATE[state_key] = message.id
        save_state()

    return message


async def post_all_embeds():
    if DATA is None:
        return

    role_ping = "||<@&1516468991424593921>||"

    if SEEDS_CHANNEL_ID > 0:
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_instock", build_instock_embed("seeds", DATA), content=role_ping)
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_next", build_next_restock_embed("seeds", DATA), content="")
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_notable", build_notable_embed("seeds", DATA), content="")
    if GEARS_CHANNEL_ID > 0:
        await upsert_embed(GEARS_CHANNEL_ID, "gears_instock", build_instock_embed("gears", DATA), content=role_ping)
        await upsert_embed(GEARS_CHANNEL_ID, "gears_next", build_next_restock_embed("gears", DATA), content="")
        await upsert_embed(GEARS_CHANNEL_ID, "gears_notable", build_notable_embed("gears", DATA), content="")
    if WEATHER_CHANNEL_ID > 0:
        await upsert_embed(WEATHER_CHANNEL_ID, "weather", build_weather_embed(DATA), content=role_ping)


@tasks.loop(seconds=AUTO_POST_INTERVAL_SECONDS)
async def auto_post_loop():
    try:
        await post_all_embeds()
    except Exception as exc:
        print(f"Auto post failed: {exc}")


@tasks.loop(seconds=60)
async def refresh_embeds_loop():
    try:
        await post_all_embeds()
    except Exception as exc:
        print(f"Refresh embeds failed: {exc}")


@bot.event
async def on_ready():
    global DATA
    if DATA is None:
        DATA = load_data()
    load_state()
    print(f"Logged in as {bot.user}")
    print(f"SEEDS_CHANNEL_ID={SEEDS_CHANNEL_ID} GEARS_CHANNEL_ID={GEARS_CHANNEL_ID} WEATHER_CHANNEL_ID={WEATHER_CHANNEL_ID}")
    if not auto_post_loop.is_running():
        auto_post_loop.start()
    if not refresh_embeds_loop.is_running():
        refresh_embeds_loop.start()


@bot.command(name="seeds")
async def seeds_command(ctx):
    if SEEDS_CHANNEL_ID <= 0:
        await ctx.reply("SEEDS_CHANNEL_ID is not configured.", mention_author=False)
        return
    role_ping = "||<@&1516468991424593921>||"
    await upsert_embed(SEEDS_CHANNEL_ID, "seeds_instock", build_instock_embed("seeds", DATA), content=role_ping)
    await upsert_embed(SEEDS_CHANNEL_ID, "seeds_next", build_next_restock_embed("seeds", DATA), content="")
    await upsert_embed(SEEDS_CHANNEL_ID, "seeds_notable", build_notable_embed("seeds", DATA), content="")
    await ctx.reply(f"Seed shop embeds posted to <#{SEEDS_CHANNEL_ID}>.", mention_author=False)


@bot.command(name="gears")
async def gears_command(ctx):
    if GEARS_CHANNEL_ID <= 0:
        await ctx.reply("GEARS_CHANNEL_ID is not configured.", mention_author=False)
        return
    role_ping = "||<@&1516468991424593921>||"
    await upsert_embed(GEARS_CHANNEL_ID, "gears_instock", build_instock_embed("gears", DATA), content=role_ping)
    await upsert_embed(GEARS_CHANNEL_ID, "gears_next", build_next_restock_embed("gears", DATA), content="")
    await upsert_embed(GEARS_CHANNEL_ID, "gears_notable", build_notable_embed("gears", DATA), content="")
    await ctx.reply(f"Gear shop embeds posted to <#{GEARS_CHANNEL_ID}>.", mention_author=False)


@bot.command(name="weather")
async def weather_command(ctx):
    if WEATHER_CHANNEL_ID <= 0:
        await ctx.reply("WEATHER_CHANNEL_ID is not configured.", mention_author=False)
        return
    await upsert_embed(WEATHER_CHANNEL_ID, "weather", build_weather_embed(DATA), content="||<@&1516468991424593921>||")
    await ctx.reply(f"Weather embed posted to <#{WEATHER_CHANNEL_ID}>.", mention_author=False)


@bot.command(name="postall")
async def post_all_command(ctx):
    sent = []
    role_ping = "||<@&1516468991424593921>||"
    if SEEDS_CHANNEL_ID > 0:
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_instock", build_instock_embed("seeds", DATA), content=role_ping)
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_next", build_next_restock_embed("seeds", DATA), content="")
        await upsert_embed(SEEDS_CHANNEL_ID, "seeds_notable", build_notable_embed("seeds", DATA), content="")
        sent.append(f"Seeds -> <#{SEEDS_CHANNEL_ID}>")
    if GEARS_CHANNEL_ID > 0:
        await upsert_embed(GEARS_CHANNEL_ID, "gears_instock", build_instock_embed("gears", DATA), content=role_ping)
        await upsert_embed(GEARS_CHANNEL_ID, "gears_next", build_next_restock_embed("gears", DATA), content="")
        await upsert_embed(GEARS_CHANNEL_ID, "gears_notable", build_notable_embed("gears", DATA), content="")
        sent.append(f"Gears -> <#{GEARS_CHANNEL_ID}>")
    if WEATHER_CHANNEL_ID > 0:
        await upsert_embed(WEATHER_CHANNEL_ID, "weather", build_weather_embed(DATA), content=role_ping)
        sent.append(f"Weather -> <#{WEATHER_CHANNEL_ID}>")

    if not sent:
        await ctx.reply("No target channels configured.", mention_author=False)
        return

    await ctx.reply("Posted embeds to: " + ", ".join(sent), mention_author=False)


if TOKEN is None:
    print("ERROR: Set DISCORD_TOKEN or TOKEN environment variable before running the bot.")
else:
    DATA = load_data()
    bot.run(TOKEN)
