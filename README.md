======================================
# DISCORD BOT MEGA PRO (MEE6 STYLE)
# ======================================
# â ï¸ FULL SYSTEM:
# - Slash commands
# - Casino full (slot, coinflip)
# - Shop button UI
# - Auto role
# - Dashboard chá»n server
# - Login Discord OAuth2
# - Biá»ƒu Ä‘á»“ thá»‘ng kĂª
# - Donate webhook (cĂ³ thá»ƒ tĂ­ch há»£p Momo tháº­t sau)

# CĂ i:
# pip install discord.py flask requests matplotlib

import discord
from discord.ext import commands
from discord import app_commands
import json, random, time
from flask import Flask, request, redirect, session
import requests, threading
import matplotlib.pyplot as plt

TOKEN = "YOUR_BOT_TOKEN"
CLIENT_ID = "YOUR_CLIENT_ID"
CLIENT_SECRET = "YOUR_CLIENT_SECRET"
REDIRECT_URI = "http://localhost:3000/callback"

intents = discord.Intents.all()
bot = commands.Bot(command_prefix="!", intents=intents)

# ==========================
# DATABASE
# ==========================
def load():
    try:
        return json.load(open("data.json"))
    except:
        return {}

def save():
    json.dump(db, open("data.json","w"), indent=4)


db = load()

def get(uid):
    uid=str(uid)
    if uid not in db:
        db[uid] = {"cash":0,"win":0,"lose":0}
    return db[uid]

# ==========================
# DISCORD BOT
# ==========================
@bot.event
async def on_ready():
    await bot.tree.sync()
    print("READY")

# WALLET
@bot.tree.command(name="wallet")
async def wallet(i:discord.Interaction):
    u=get(i.user.id)
    e=discord.Embed(title="đŸ’° Wallet",color=0x00ffcc)
    e.add_field(name="Cash",value=u['cash'])
    await i.response.send_message(embed=e)

# COINFLIP
@bot.tree.command(name="coinflip")
async def coinflip(i:discord.Interaction,bet:int):
    u=get(i.user.id)
    if u['cash']<bet:
        return await i.response.send_message("âŒ thiáº¿u tiá»n")
    if random.choice([True,False]):
        u['cash']+=bet
        u['win']+=1
        msg="WIN"
    else:
        u['cash']-=bet
        u['lose']+=1
        msg="LOSE"
    save()
    await i.response.send_message(f"đŸª™ {msg} {bet}")

# SLOT
@bot.tree.command(name="slot")
async def slot(i:discord.Interaction,bet:int):
    u=get(i.user.id)
    if u['cash']<bet:
        return await i.response.send_message("âŒ thiáº¿u tiá»n")
    r=[random.choice(["đŸ’","đŸ’","đŸ‹"]) for _ in range(3)]
    if r[0]==r[1]==r[2]:
        win=bet*3
        u['cash']+=win
        u['win']+=1
        msg=f"WIN {win}"
    else:
        u['cash']-=bet
        u['lose']+=1
        msg=f"LOSE {bet}"
    save()
    await i.response.send_message(f"đŸ° {' '.join(r)} {msg}")

# SHOP BUTTON
class ShopView(discord.ui.View):
    @discord.ui.button(label="VIP",style=discord.ButtonStyle.green)
    async def vip(self,i:discord.Interaction,b):
        u=get(i.user.id)
        if u['cash']<500:
            return await i.response.send_message("âŒ thiáº¿u tiá»n")
        u['cash']-=500
        role=discord.utils.get(i.guild.roles,name="VIP")
        if role:
            await i.user.add_roles(role)
        save()
        await i.response.send_message("âœ… mua VIP")

@bot.tree.command(name="shop")
async def shop(i:discord.Interaction):
    await i.response.send_message("đŸ›’ Shop",view=ShopView())

# ==========================
# WEB DASHBOARD
# ==========================
app=Flask(__name__)
app.secret_key="secret"

@app.route('/')
def home():
    if 'user' in session:
        return f"Hello {session['user']['username']}<br><a href='/servers'>Servers</a>"
    return "<a href='/login'>Login Discord</a>"

@app.route('/login')
def login():
    return redirect(f"https://discord.com/api/oauth2/authorize?client_id={CLIENT_ID}&redirect_uri={REDIRECT_URI}&response_type=code&scope=identify guilds")

@app.route('/callback')
def callback():
    code=request.args.get('code')
    data_token={
        'client_id':CLIENT_ID,
        'client_secret':CLIENT_SECRET,
        'grant_type':'authorization_code',
        'code':code,
        'redirect_uri':REDIRECT_URI
    }
    r=requests.post('https://discord.com/api/oauth2/token',data=data_token,headers={'Content-Type':'application/x-www-form-urlencoded'})
    token=r.json()['access_token']
    user=requests.get('https://discord.com/api/users/@me',headers={'Authorization':f'Bearer {token}'}).json()
    guilds=requests.get('https://discord.com/api/users/@me/guilds',headers={'Authorization':f'Bearer {token}'}).json()
    session['user']=user
    session['guilds']=guilds
    return redirect('/servers')

@app.route('/servers')
def servers():
    if 'guilds' not in session:
        return redirect('/')
    html="<h1>Servers</h1>"
    for g in session['guilds']:
        html+=f"<p>{g['name']}</p>"
    return html

# ==========================
# STATS CHART
# ==========================
@app.route('/stats/<uid>')
def stats(uid):
    u=get(uid)
    labels=["win","lose"]
    values=[u['win'],u['lose']]
    plt.bar(labels,values)
    plt.savefig("chart.png")
    return open("chart.png","rb").read()

# ==========================
# DONATE WEBHOOK (MOMO READY)
# ==========================
@app.route('/donate',methods=['POST'])
def donate():
    uid=request.json.get('user_id')
    amount=request.json.get('amount')
    u=get(uid)
    u['cash']+=amount
    save()
    return {"ok":True}

# ==========================
# RUN
# ==========================
def web():
    app.run(host="0.0.0.0",port=3000)

threading.Thread(target=web).start()
