import asyncio
import logging
import random
import os
from typing import Dict, Set, Optional, Any, List
from dataclasses import dataclass
from enum import Enum

import telegram
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, CallbackQueryHandler,
    ContextTypes
)

# Logging setup
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Config
BOT_TOKEN = os.getenv('BOT_TOKEN', '7720847875:AAEp4lX9UM7P5iApIJX_ppvIHJgYn0d0eL8')

# GÖRSEL URL'leri - GÜNCELLENMİŞ
IMAGES = {
    "START": "https://images.unsplash.com/photo-1518709268805-4e9042af2176?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80",
    "VAMPIR_WIN": "https://images.unsplash.com/photo-1573148164257-8a2b173be464?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80",
    "KOYLU_WIN": "https://images.unsplash.com/photo-1506905925346-21bda4d32df4?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80",
    "KURT": "https://images.unsplash.com/photo-1514984879728-be0aff75a6e8?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80",
    "ROMANTIC": "https://images.unsplash.com/photo-1518568814500-bf0f8d125f46?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80",
    "STEAMY": "https://images.unsplash.com/photo-1516487106395-f3c55b3b6e0e?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1200&q=80"
}

# EĞLENCELİ KÖYLÜ LAKAPLARI - GÜNCELLENMİŞ
KOYLU_LAKAPLARI = [
    "👨‍🌾 Köyün Muhtarı",
    "👩‍🌾 Köyün Güzeli", 
    "🧑‍🌾 Yaramaz Çocuk",
    "👨‍🌾 Bilge Çiftçi",
    "👩‍🌾 Dedikoducu Kadın",
    "🧑‍🌾 Köy Delisi",
    "👨‍🌾 Kasap Usta",
    "👩‍🌾 Fırıncı Kadın",
    "🧑‍🌾 Avcı Mehmet",
    "👨‍🌾 Balıkçı Hasan",
    "👩‍🌾 Öğretmen Ayşe",
    "🧑‍🌾 Doktor Yardımcısı",
    "👨‍🌾 Demirci Usta",
    "👩‍🌾 Çamaşırcı Kadın",
    "🧑‍🌾 Çoban Ali",
    "😈 Köyün Sapığı",
    "🔥 Köyün Yaramaz Kızı"
]

# Global games dictionary
games: Dict[int, 'GameState'] = {}
state_lock = asyncio.Lock()

@dataclass
class GameConfig:
    group_id: Optional[int] = None
    started_by: Optional[int] = None

class GamePhase(Enum):
    LOBBY = "lobby"
    PLAYING = "playing"
    NIGHT = "night"
    DAY = "day"

@dataclass
class Player:
    user_id: int
    username: str
    role: Optional[str] = None
    alive: bool = True
    lakap: Optional[str] = None

# Role constants - GÜNCELLENMİŞ
ROLES = {
    "VAMPIR": "🧛 Vampir",
    "DOKTOR": "🩺 Doktor", 
    "KOYLU": "👨‍🌾 Köylü",
    "KURT": "🐺 Alfa Kurt",
    "SAPIK": "😈 Köyün Sapığı",
    "YARAMAZ_KIZ": "🔥 Köyün Yaramaz Kızı"
}

class GameState:
    def __init__(self):
        self._reset()
    
    def _reset(self):
        self.phase: GamePhase = GamePhase.LOBBY
        self.players: Dict[int, Player] = {}
        self.config = GameConfig()
        self.dead: Set[int] = set()
        self.night_actions: Dict[str, Any] = {
            "vampire": {}, 
            "doctor": None, 
            "kurt": None,
            "sapik": None,
            "yaramaz_kiz": None
        }
        self.votes: Dict[int, int] = {}
        self.expected_voters: Set[int] = set()
        self._timer_task: Optional[asyncio.Task] = None
        self._join_timer_task: Optional[asyncio.Task] = None
        self.join_time_left: int = 60
        self.vote_message_id: Optional[int] = None
        self._game_active: bool = False
        self.join_message_id: Optional[int] = None
        self.night_button_messages: Dict[int, int] = {}
        self.extra_time_used: bool = False
        self.extra_time_votes: Set[int] = set()

    @property
    def group_id(self) -> Optional[int]:
        return self.config.group_id
    
    @group_id.setter
    def group_id(self, value: int):
        self.config.group_id = value

    @property
    def started_by(self) -> Optional[int]:
        return self.config.started_by
    
    @started_by.setter
    def started_by(self, value: int):
        self.config.started_by = value

    def reset(self):
        """Clean up tasks before reset"""
        if self._timer_task and not self._timer_task.done():
            self._timer_task.cancel()
        if self._join_timer_task and not self._join_timer_task.done():
            self._join_timer_task.cancel()
        
        self._reset()
        self.phase = GamePhase.LOBBY
        self._game_active = False
        self.players = {}
        self.dead = set()
        self.night_actions = {"vampire": {}, "doctor": None, "kurt": None, "sapik": None, "yaramaz_kiz": None}
        self.votes = {}
        self.expected_voters = set()
        self.vote_message_id = None
        self.group_id = None
        self.started_by = None
        self.join_message_id = None
        self.night_button_messages.clear()
        self.extra_time_used = False
        self.extra_time_votes.clear()
        logger.info("🛑 Oyun tamamen resetlendi!")

    def is_active(self) -> bool:
        return self._game_active

    def set_active(self, active: bool):
        self._game_active = active

    def add_player(self, user_id: int, username: str) -> bool:
        """Add player if not already in game"""
        if user_id in self.players:
            return False
        self.players[user_id] = Player(user_id, username)
        return True

    def get_alive_players(self) -> list:
        return [p for p in self.players.values() if p.alive]

    def kill_player(self, user_id: int):
        if user_id in self.players:
            self.players[user_id].alive = False
            self.dead.add(user_id)

    async def add_extra_time(self, minutes: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
        """Sadece oyunu başlatan kişi extra süre ekleyebilir"""
        if self.extra_time_used:
            return False
            
        if minutes <= 0 or minutes > 10:
            return False
            
        self.join_time_left += minutes * 60
        self.extra_time_used = True
        
        await safe_send_message(
            context, self.group_id,
            f"⏰ *EXTRA SÜRE EKLENDİ!*\n\n"
            f"➕ {minutes} dakika ek süre eklendi!\n"
            f"⏳ Yeni toplam süre: {self.join_time_left} saniye\n"
            f"🎮 Oyunu başlatan tarafından eklendi"
        )
        return True

    def assign_roles(self):
        """Assign roles to players - YENİ ROLLER EKLENDİ"""
        alive_players = list(self.players.values())
        random.shuffle(alive_players)
        
        player_count = len(alive_players)
        
        # Vampir sayısını belirle
        if player_count <= 6:
            vampire_count = 1
        elif player_count <= 12:
            vampire_count = 2
        else:
            vampire_count = 3
        
        # 8+ oyuncuda özel roller ekle
        has_kurt = player_count >= 10
        has_special_roles = player_count >= 8
        
        roles_to_assign = []
        
        # Vampirler
        for _ in range(vampire_count):
            roles_to_assign.append(ROLES["VAMPIR"])
        
        # Doktor
        roles_to_assign.append(ROLES["DOKTOR"])
        
        # Alfa Kurt (10+ oyuncuda)
        if has_kurt:
            roles_to_assign.append(ROLES["KURT"])
        
        # YENİ: Özel Roller (8+ oyuncuda)
        if has_special_roles:
            # Sapık veya Yaramaz Kız'dan birini seç
            special_roles = [ROLES["SAPIK"], ROLES["YARAMAZ_KIZ"]]
            roles_to_assign.append(random.choice(special_roles))
        
        # Köylüler (kalanlar)
        koylu_count = player_count - len(roles_to_assign)
        koylu_lakaplari = random.sample(KOYLU_LAKAPLARI, min(koylu_count, len(KOYLU_LAKAPLARI))
        
        for i in range(koylu_count):
            if i < len(koylu_lakaplari):
                roles_to_assign.append(koylu_lakaplari[i])
            else:
                roles_to_assign.append(ROLES["KOYLU"])
        
        # Rolleri karıştır ve dağıt
        random.shuffle(roles_to_assign)
        for player, role in zip(alive_players, roles_to_assign):
            player.role = role
            if role in KOYLU_LAKAPLARI or role == ROLES["KOYLU"]:
                player.lakap = role

# Application instance
app = None

def get_game(group_id: int) -> GameState:
    """Belirli grup için GameState al"""
    if group_id not in games:
        games[group_id] = GameState()
        logger.info(f"🎮 Yeni oyun instance'ı oluşturuldu: {group_id}")
    return games[group_id]

async def safe_send_message(
    context: ContextTypes.DEFAULT_TYPE, 
    chat_id: int, 
    text: str, 
    reply_markup: Optional[InlineKeyboardMarkup] = None,
    parse_mode: str = "Markdown"
) -> bool:
    """Safely send message with error handling"""
    try:
        await context.bot.send_message(
            chat_id=chat_id, 
            text=text, 
            reply_markup=reply_markup, 
            parse_mode=parse_mode
        )
        return True
    except Exception as e:
        logger.error(f"Message send error to {chat_id}: {e}")
        return False

async def safe_send_photo(
    context: ContextTypes.DEFAULT_TYPE,
    chat_id: int,
    photo_url: str,
    caption: str = "",
    parse_mode: str = "Markdown"
) -> bool:
    """Güvenli fotoğraf gönder"""
    try:
        await context.bot.send_photo(
            chat_id=chat_id,
            photo=photo_url,
            caption=caption,
            parse_mode=parse_mode
        )
        return True
    except Exception as e:
        logger.error(f"Photo send error to {chat_id}: {e}")
        await safe_send_message(context, chat_id, caption, parse_mode=parse_mode)
        return False

async def safe_send_pm(
    user_id: int, 
    text: str, 
    reply_markup: Optional[InlineKeyboardMarkup] = None
) -> bool:
    """Safely send private message"""
    global app
    if app is None:
        return False
    try:
        await app.bot.send_message(
            chat_id=user_id, 
            text=text, 
            reply_markup=reply_markup,
            parse_mode="Markdown"
        )
        return True
    except Exception as e:
        logger.error(f"PM send error to {user_id}: {e}")
        return False

async def send_mention(
    context: ContextTypes.DEFAULT_TYPE, 
    chat_id: int, 
    user_id: int, 
    text: str
) -> bool:
    """Send message with user mention"""
    try:
        game = get_game(chat_id)
        player_name = next(
            (p.username for p in game.players.values() if p.user_id == user_id), 
            "Bilinmeyen"
        )
        mention = f"[{player_name}](tg://user?id={user_id})"
        await safe_send_message(
            context, chat_id, f"{mention} {text}", parse_mode="Markdown"
        )
        return True
    except Exception as e:
        logger.error(f"Mention error: {e}")
        return False

async def send_death_notification(context: ContextTypes.DEFAULT_TYPE, game: GameState, player_id: int, death_type: str, killer_info: str = ""):
    """Ölen oyuncuya ölüm bildirimi gönder"""
    player = game.players.get(player_id)
    if not player:
        return
    
    death_messages = {
        "vampire_night": "🧛‍♂️ *VAMPİRLER TARAFINDAN ÖLDÜRÜLDÜN!*\n\n🌙 Gece vakti vampirler seni buldu...\n🩸 Kanını emerek seni öldürdüler!\n💀 Artık bir hayaletsin!",
        
        "lynch_day": "👨‍🌾 *KÖYLÜLER TARAFINDAN LİNÇ EDİLDİN!*\n\n☀️ Gündüz yapılan oylamada...\n🗳️ Köylüler seni vampir sanıp linç etti!\n💀 Artık bir hayaletsin!",
        
        "kurt_kill": "🐺 *ALFA KURT TARAFINDAN ÖLDÜRÜLDÜN!*\n\n🌙 Gece vakti kurt seni avladı...\n⚔️ Kurt sürüsüne yem oldun!\n💀 Artık bir hayaletsin!",
        
        "doctor_save": "🩺 *DOKTOR TARAFINDAN KURTARILDIN!*\n\n🌙 Vampirler seni ısırmıştı...\n💉 Doktor zamanında müdahale etti!\n❤️ Hayattasın, şanslısın!",
        
        "protected": "⛑️ *KORUYUCU TARAFINDAN KORUNDUN!*\n\n🌙 Gece saldırısından...\n🛡️ Koruyucu melek seni korudu!\n❤️ Hayattasın!"
    }
    
    message = death_messages.get(death_type, 
        "💀 *OYUNDAN ELENDİN!*\n\nArtık bir hayaletsin. Bir sonraki oyunu bekle!")
    
    if killer_info:
        message += f"\n\n🔍 *Detay:* {killer_info}"
    
    message += f"\n\n🎭 *Rolün:* {player.role}"
    
    await safe_send_pm(player_id, message)

async def send_romantic_notification(context: ContextTypes.DEFAULT_TYPE, game: GameState, visitor_id: int, target_id: int, role_type: str):
    """Ateşli ziyaret bildirimi gönder"""
    visitor = game.players[visitor_id]
    target = game.players[target_id]
    
    romantic_messages = {
        "sapik": {
            "visitor_msg": (
                "😈 *GECE ZİYARETİN BAŞARILI!*\n\n"
                "🌙 Karanlıkta sessizce süzülerek...\n"
                f"🔥 {target.username}'in odasına girdin!\n"
                "💕 Onu tatlı rüyalarından uyandırdın...\n"
                "🛏️ Sabaha kadar unutulmaz anlar yaşadınız!\n\n"
                "🎯 Hedefin bu geceyi asla unutamayacak! 😉"
            ),
            "target_msg": (
                "🔥 *GECE ZİYARETÇİN VAR!*\n\n"
                "🌙 Derin uykundayken...\n"
                f"😈 {visitor.username} odana sessizce girdi!\n"
                "💕 Uyandığın anda ateşli bakışlarıyla karşılaştın...\n"
                "🛏️ Sabaha kadar tutkulu anlar yaşadınız!\n\n"
                "😳 Bu geceyi asla unutamayacaksın! 💋"
            ),
            "image": IMAGES["STEAMY"]
        },
        "yaramaz_kiz": {
            "visitor_msg": (
                "🔥 *GECE MACERAN BAŞLADI!*\n\n"
                "🌙 Gecenin karanlığını yırtarak...\n"
                f"💃 {target.username}'in kapısını çaldın!\n"
                "❤️ İçeri girdiğin an elektrik çaktı...\n"
                "🎉 Bütün gece tutkulu danslar ettiniz!\n\n"
                "💋 Hedefin bu geceye bayılacak! 😘"
            ),
            "target_msg": (
                "💃 *SÜRPRİZ ZİYARET!*\n\n"
                "🌙 Gecenin sessizliğinde...\n"
                f"🔥 {visitor.username} kapını çaldı!\n"
                "❤️ Ateşli bakışlarıyla içeri davet ettin...\n"
                "💋 Sabaha kadar romantik anlar yaşadınız!\n\n"
                "😍 Bu gece hayatının en güzel sürprizi oldu! 🌹"
            ),
            "image": IMAGES["ROMANTIC"]
        }
    }
    
    message_data = romantic_messages.get(role_type)
    if not message_data:
        return
    
    await safe_send_photo(
        context, visitor_id, 
        message_data["image"],
        message_data["visitor_msg"]
    )
    
    await safe_send_photo(
        context, target_id,
        message_data["image"], 
        message_data["target_msg"]
    )

def build_player_buttons(game: GameState, only_alive: bool = True, group_id: int = None, phase: str = "night") -> Optional[InlineKeyboardMarkup]:
    """Build inline keyboard with player buttons"""
    if not game.players:
        return None
    
    buttons = []
    player_list = game.get_alive_players() if only_alive else list(game.players.values())
    
    row = []
    for player in player_list:
        if not only_alive and not player.alive:
            continue
            
        button = InlineKeyboardButton(
            f"{player.username} {'💀' if not player.alive else ''}", 
            callback_data=f"target_{group_id}_{player.user_id}_{phase}"
        )
        row.append(button)
        
        if len(row) == 2:
            buttons.append(row)
            row = []
    
    if row:
        buttons.append(row)
    
    return InlineKeyboardMarkup(buttons) if buttons else None

def build_join_button() -> InlineKeyboardMarkup:
    """Oyuna katıl butonu oluştur"""
    button = InlineKeyboardButton("🎮 Oyuna Katıl", callback_data="join_game")
    return InlineKeyboardMarkup([[button]])

async def update_join_message(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Katılma mesajını güncelle"""
    if not game.join_message_id:
        return
    
    try:
        player_list = "🎮 *Katılan Oyuncular:*\n"
        
        if not game.players:
            player_list += "Henüz kimse katılmadı...\n"
        else:
            for i, player in enumerate(game.players.values(), 1):
                if game.phase == GamePhase.LOBBY:
                    status = "❤️ Canlı"
                    emoji = "❤️"
                else:
                    status = "❤️ Canlı" if player.alive else "💀 Ölü"
                    emoji = "❤️" if player.alive else "💀"
                
                player_list += f"{i}. {player.username} {emoji} {status}\n"
        
        player_count = len(game.players)
        min_players = 5
        remaining = max(0, min_players - player_count)
        
        info_text = f"\n📊 *Durum:* {player_count}/{min_players} kişi"
        
        if remaining > 0:
            info_text += f" ({remaining} kişi daha gerekli)"
        else:
            info_text += " ✅ (Minimum tamamlandı!)"
        
        full_text = player_list + info_text
        
        await context.bot.edit_message_text(
            chat_id=game.group_id,
            message_id=game.join_message_id,
            text=full_text,
            reply_markup=build_join_button(),
            parse_mode="Markdown"
        )
        
    except Exception as e:
        logger.error(f"Katılma mesajı güncelleme hatası: {e}")

async def pin_join_message(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Katılma mesajını sabitle"""
    try:
        join_text = (
            "🧛‍♂️ *Vampir Köylü Oyunu Başladı!*\n\n"
            "👥 Aşağıdaki butona tıklayarak oyuna katılın!\n"
            "⚡ En az 5 kişi gerekiyor.\n"
            "⏰ 5. oyuncudan sonra 1 dakika bekleme süresi başlar.\n\n"
            "🎮 *Katılan Oyuncular:*\n"
            "Henüz kimse katılmadı..."
        )
        
        message = await context.bot.send_message(
            chat_id=game.group_id,
            text=join_text,
            reply_markup=build_join_button(),
            parse_mode="Markdown"
        )
        
        await context.bot.pin_chat_message(
            chat_id=game.group_id,
            message_id=message.message_id
        )
        game.join_message_id = message.message_id
        logger.info(f"Grup {game.group_id}: Butonlu katılma mesajı sabitlendi")
        
    except Exception as e:
        logger.error(f"Mesaj sabitleme hatası: {e}")

async def unpin_join_message(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Katılma mesajını sabitten kaldır"""
    try:
        if game.join_message_id:
            await context.bot.unpin_chat_message(
                chat_id=game.group_id,
                message_id=game.join_message_id
            )
            logger.info(f"Grup {game.group_id}: Katılma mesajı sabitten kaldırıldı")
    except Exception as e:
        logger.error(f"Sabit kaldırma hatası: {e}")

async def clear_night_buttons(game: GameState):
    """Sadece gece butonlarını temizle"""
    for player in game.get_alive_players():
        if player.user_id in game.night_button_messages:
            try:
                await safe_send_pm(
                    player.user_id,
                    "🔒 *Gece Oylaması Kapandı!*\n\n"
                    "⏰ Gece oylama süresi doldu.\n"
                    "📊 Sonuçlar açıklanıyor...\n"
                    "🌅 Gündüz hazırlıkları başlıyor!"
                )
            except Exception as e:
                logger.error(f"Gece buton uyarısı hatası {player.username}: {e}")
    
    game.night_button_messages.clear()
    logger.info(f"Grup {game.group_id}: 🌙 Gece butonları temizlendi")

# === COMMAND HANDLERS ===

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Start komutu"""
    user = update.effective_user
    
    start_text = (
        "🤖 *Vampir Köylü Botuna Hoş Geldiniz!*\n\n"
        "🎮 Bu bot gruplarda Vampir Köylü oyunu oynatır.\n\n"
        "📋 *Hızlı Başlangıç:*\n"
        "1. Gruba `/wstart` yazın\n"
        "2. Butona tıklayarak katılın\n"
        "3. Roller özelden gönderilir\n"
        "4. Vampirleri bulmaya çalışın!\n\n"
        "❓ Tüm komutlar için `/whelp` yazın.\n"
        "📚 Oyun detayları için `/wbilgi` yazın."
    )
    
    await update.message.reply_text(
        start_text,
        parse_mode="Markdown",
        reply_markup=InlineKeyboardMarkup([[
            InlineKeyboardButton("📖 Detaylı Rehber", callback_data="help_rules"),
            InlineKeyboardButton("🎮 Komutlar", callback_data="help_commands")
        ]])
    )

async def wstart(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Start new game"""
    async with state_lock:
        chat = update.effective_chat
        group_id = chat.id
        
        if chat.type not in ["group", "supergroup"]:
            await update.message.reply_text("❌ Bu komut sadece grupta kullanılabilir!")
            return
        
        game = get_game(group_id)
        
        if game.is_active():
            await update.message.reply_text("❌ Bu grupta zaten bir oyun devam ediyor!")
            return
        
        game.reset()
        game.group_id = group_id
        game.started_by = update.effective_user.id
        game.set_active(True)
        game.phase = GamePhase.LOBBY
        
        await pin_join_message(context, game)
        logger.info(f"🎮 Grup {group_id}: Oyun başlatıldı")

async def wjoin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Join game lobby"""
    user = update.effective_user
    
    if not update.effective_chat or update.effective_chat.type not in ["group", "supergroup"]:
        await update.message.reply_text("❌ Bu komut grupta /wstart ile başlatılmalı!")
        return
    
    group_id = update.effective_chat.id
    game = get_game(group_id)
    
    if game.phase != GamePhase.LOBBY or not game.is_active():
        await update.message.reply_text("⚠️ Bu grupta oyun başladı veya bitti! Katılamazsınız.")
        return
    
    if not game.add_player(user.id, user.first_name or user.username or "Bilinmeyen"):
        await update.message.reply_text("❌ Zaten bu oyundasınız!")
        return
    
    if not await safe_send_pm(user.id, "✅ Oyuna katıldınız! Roller özelden gönderilecek."):
        await update.message.reply_text(
            "❌ Bot size özel mesaj gönderemedi! Lütfen bota özelden yazın.",
            reply_markup=InlineKeyboardMarkup([[
                InlineKeyboardButton("🤖 Bota Git", url="https://t.me/Wwampir_bot")
            ]])
        )
        if user.id in game.players:
            del game.players[user.id]
        return
    
    await send_mention(context, group_id, user.id, "oyuna katıldı! 🎉")
    await update_join_message(context, game)
    
    player_count = len(game.players)
    logger.info(f"👥 Grup {group_id}: Oyuncu katıldı: {player_count} kişi")
    
    if player_count == 5:
        if game._join_timer_task and not game._join_timer_task.done():
            game._join_timer_task.cancel()
        
        game.join_time_left = 60
        game._join_timer_task = asyncio.create_task(join_countdown(context, game))
        
        await safe_send_message(
            context, group_id,
            "🎉 5 kişi tamamlandı!\n⏳ 1 dakika içinde başka oyuncu katılmazsa oyun başlayacak."
        )
    elif player_count > 5 and game._join_timer_task and not game._join_timer_task.done():
        game.join_time_left = 60
        await safe_send_message(
            context, group_id,
            f"➕ Yeni oyuncu! Süre 60 saniyeye sıfırlandı.\n👥 Toplam: {player_count} oyuncu"
        )

async def join_countdown(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Countdown for lobby phase"""
    group_id = game.group_id
    while game.join_time_left > 0 and game.phase == GamePhase.LOBBY:
        await asyncio.sleep(1)
        game.join_time_left -= 1
        
        if game.join_time_left == 30:
            await safe_send_message(context, group_id, "⚠️ 30 saniye kaldı! Katılacak yeni oyuncu yoksa oyun başlayacak.")
    
    if len(game.players) >= 5 and game.phase == GamePhase.LOBBY:
        await start_game(context, game)
    else:
        await safe_send_message(context, group_id, "❌ Yeterli oyuncu yok! Oyun iptal edildi.")
        game.reset()

async def wson(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Stop/cancel game"""
    user_id = update.effective_user.id
    
    if not update.effective_chat or update.effective_chat.type not in ["group", "supergroup"]:
        await update.message.reply_text("❌ Bu komut grupta kullanılabilir!")
        return
    
    group_id = update.effective_chat.id
    game = get_game(group_id)
    
    if game.started_by != user_id:
        await update.message.reply_text("❌ Sadece oyunu başlatan kişi oyunu iptal edebilir!")
        return
    
    game.reset()
    await update.message.reply_text("🛑 Oyun iptal edildi!")
    logger.info(f"Grup {group_id}: Oyun iptal edildi")

async def whelp(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Show help message"""
    help_text = (
        "🧛‍♂️ *Vampir Köylü - Komutlar*\n\n"
        "🎮 **Oyun Yönetimi:**\n"
        "• `/wstart` - Oyunu başlatır (sadece grupta)\n"
        "• `/wjoin` - Oyuna katılır\n"
        "• `/wson` - Oyunu iptal eder (sadece başlatan)\n"
        "• `/wextend <dakika>` - Extra süre ekler (sadece başlatan)\n\n"
        
        "📋 **Bilgi:**\n"
        "• `/wbilgi` - Detaylı oyun rehberi ve örnekler\n"
        "• `/whelp` - Bu yardım mesajı\n\n"
        
        "⚙️ **Oyun Özellikleri:**\n"
        "• En az 5 oyuncu gerekir\n"
        "• 5. oyuncudan sonra 60 saniye bekleme\n"
        "• Oyun başlatan extra süre ekleyebilir\n"
        "• Gece: 60 saniye aksiyon süresi\n"
        "• Gündüz: 90 saniye tartışma + 30 saniye oylama\n"
        "• Inline butonlarla oylama\n"
        "• Her grupta ayrı oyun!\n\n"
        "❓ Sorularınız için oyunu başlatan kişiye yazın!"
    )
    
    await update.message.reply_text(help_text, parse_mode="Markdown")

async def wbilgi(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Detaylı oyun bilgisi ve örnekler"""
    info_text = (
        "🧛‍♂️ *VAMPİR KÖYLÜ - DETAYLI OYUN REHBERİ*\n\n"
        
        "🎯 *OYUNUN AMACI:*\n"
        "• 🧛 **Vampirler**: Tüm köylüleri öldürerek kazanır\n"
        "• 👨‍🌾 **Köylüler**: Tüm vampirleri bulup linç ederek kazanır\n\n"
        
        "👥 *ROLLER ve GÖREVLER:*\n"
        "• 🧛 *Vampir*: Gece birini ısırır, takım arkadaşını seçemez\n"
        "• 🩺 *Doktor*: Gece birini korur, ısırılmaktan kurtarır\n"
        "• 🐺 *Alfa Kurt*: Gece birini avlar (sadece vampirleri öldürebilir)\n"
        "• 😈 *Köyün Sapığı*: Gece birinin koynuna girer\n"
        "• 🔥 *Köyün Yaramaz Kızı*: Gece birini ziyaret eder\n"
        "• 👨‍🌾 *Köylü*: Gündüz tartışır, vampirleri bulmaya çalışır\n\n"
        
        "🔄 *OYUN DÖNGÜSÜ:*\n"
        "1. 🌙 *GECE* (60 saniye)\n"
        "   - Vampirler avlanır\n"
        "   - Doktor koruma yapar\n"
        "   - Kurt ava çıkar\n"
        "   - Sapık/Yaramaz Kız ziyaret eder\n"
        "2. ☀️ *GÜNDÜZ* (90s tartışma + 30s oylama)\n"
        "   - Köylüler tartışır\n"
        "   - Şüphelerinizi paylaşın\n"
        "   - Oylama ile birini linç ederler\n\n"
        
        "🎭 *ÖRNEK OYUN AKIŞI:*\n"
        "```\n"
        "🌙 1. GECE:\n"
        "- Vampir: Ali'yi ısırdı\n"
        "- Doktor: Ayşe'yi korudu\n"
        "- Ali doktordan korundu → KURTULDU!\n"
        "```\n"
        "```\n"
        "☀️ 1. GÜNDÜZ:\n"
        "- Köylüler tartışıyor...\n"
        "- Mehmet: 'Bence Ahmet şüpheli!'\n"
        "- Oylama: Ahmet 3 oy aldı → LİNÇ EDİLDİ!\n"
        "- Ahmet: 🧛 Vampir çıktı!\n"
        "```\n"
        "```\n"
        "🌙 2. GECE:\n"
        "- Vampir: Mehmet'i ısırdı\n"
        "- Doktor: kimseyi korumadı\n"
        "- Mehmet öldü 💀\n"
        "```\n\n"
        
        "🏆 *KAZANMA KOŞULLARI:*\n"
        "• 🧛 *Vampirler kazanır*: Canlı vampir ≥ canlı köylü\n"
        "• 👨‍🌾 *Köylüler kazanır*: Tüm vampirler ölü\n\n"
        
        "⚡ *STRATEJİ İPUÇLARI:*\n"
        "• 🧛 Vampir: Takım arkadaşınla koordineli saldır!\n"
        "• 🩺 Doktor: Kimin saldırıya uğrayacağını tahmin et!\n"
        "• 🐺 Kurt: Vampirleri bulmaya odaklan!\n"
        "• 😈🔊 Sapık/Yaramaz Kız: Eğlenceyi getir!\n"
        "• 👨‍🌾 Köylü: Davranışları gözle, tutarsızlıkları bul!\n\n"
        
        "🎮 *KOMUTLAR:* `/whelp` yazarak tüm komutları görebilirsiniz."
    )
    
    await update.message.reply_text(info_text, parse_mode="Markdown")

async def wnasıloynanır(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Eski komut - yeni komuta yönlendir"""
    await update.message.reply_text(
        "ℹ️ *Komut Güncellendi!*\n\n"
        "Artık oyun bilgileri için `/wbilgi` komutunu kullanın.\n"
        "Detaylı oyun rehberi ve örnekler için yazın: `/wbilgi`",
        parse_mode="Markdown"
    )

async def wextend(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Oyunu başlatan kişi extra süre ekler"""
    user_id = update.effective_user.id
    group_id = update.effective_chat.id
    
    if group_id not in games:
        await update.message.reply_text("❌ Aktif oyun bulunamadı!")
        return
    
    game = games[group_id]
    
    if game.started_by != user_id:
        await update.message.reply_text("❌ Sadece oyunu başlatan kişi extra süre ekleyebilir!")
        return
    
    if game.phase != GamePhase.LOBBY:
        await update.message.reply_text("❌ Sadece lobi aşamasında extra süre eklenebilir!")
        return
    
    if not context.args or len(context.args) != 1:
        await update.message.reply_text("❌ Kullanım: `/wextend <dakika>`\nÖrnek: `/wextend 3`")
        return
    
    try:
        minutes = int(context.args[0])
        if minutes <= 0 or minutes > 10:
            await update.message.reply_text("❌ 1-10 dakika arası süre ekleyebilirsiniz!")
            return
            
        success = await game.add_extra_time(minutes, context)
        if success:
            await update.message.reply_text(f"✅ {minutes} dakika extra süre eklendi!")
        else:
            await update.message.reply_text("❌ Extra süre zaten kullanıldı!")
            
    except ValueError:
        await update.message.reply_text("❌ Geçersiz sayı! Örnek: `/wextend 3`")

# === GAME LOGIC ===

async def start_game(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Start the actual game - GÜNCELLENMİŞ"""
    if len(game.players) < 5:
        await safe_send_message(context, game.group_id, "❌ Yeterli oyuncu yok! Oyun başlatılamadı.")
        game.reset()
        return
    
    # YENİ: Sabiti kaldır
    await unpin_join_message(context, game)
    
    await safe_send_photo(
        context,
        game.group_id,
        IMAGES["START"],
        "🎬 *Oyun Başladı!*\n\n🎭 Roller özelden gönderildi.\n🌙 İlk gece başlıyor..."
    )
    
    game.assign_roles()
    game.phase = GamePhase.PLAYING
    
    logger.info(f"Grup {game.group_id}: Roller dağıtıldı!")
    
    failed_pms = []
    for player in game.players.values():
        role_msg = f"🎭 *Rolün: {player.role}*\n\n"
        
        if player.lakap and player.lakap != ROLES["KOYLU"]:
            role_msg += f"🏷️ *Lakabın:* {player.lakap}\n\n"
        
        takim_arkadaslari = []
        if "Vampir" in player.role:
            takim_arkadaslari = [p for p in game.players.values() if "Vampir" in p.role and p.user_id != player.user_id]
            role_msg += "🧛 *Takım Arkadaşların:* "
            if takim_arkadaslari:
                role_msg += ", ".join([p.username for p in takim_arkadaslari])
            else:
                role_msg += "Tek vampir sensin!"
            role_msg += "\n\n🌑 *Gece:* Birini ısıracaksın!\n⚠️ Takım arkadaşını seçemezsin."
        elif "Doktor" in player.role:
            role_msg += "🩺 *Takımın:* Köylüler\n\n💉 *Gece:* Birini koruyabilirsin!\n⚠️ Takım arkadaşını seçemezsin."
        elif "Kurt" in player.role:
            role_msg += "🐺 *Takımın:* Köylüler\n\n🐺 *Gece:* Birini avlayabilirsin!\n🎯 Sadece wampirleri öldürebilirsin."
        elif player.role == ROLES["SAPIK"]:
            role_msg += "😈 *Takımın:* Köylüler\n\n🌙 *Gece:* Birinin koynuna girebilirsin!\n💕 Romantik bir sürpriz yap, ilişkileri geliştir!\n⚠️ Takım arkadaşını seçemezsin."
        elif player.role == ROLES["YARAMAZ_KIZ"]:
            role_msg += "🔥 *Takımın:* Köylüler\n\n🌙 *Gece:* Birini ziyaret edebilirsin!\n💃 Ateşli bir sürpriz yap, eğlenceyi getir!\n⚠️ Takım arkadaşını seçemezsin."
        else:
            role_msg += "👨‍🌾 *Takımın:* Köylüler\n\n👨‍🌾 *Gündüz:* Vampirleri bulmaya çalış!\n🗳️ Oylama ile şüpheliyi linç et!"
        
        if not await safe_send_pm(player.user_id, role_msg):
            failed_pms.append(player.username)
    
    if failed_pms:
        await safe_send_message(context, game.group_id, f"⚠️ Roller şu kişilere ulaşılamadı: {', '.join(failed_pms)}")
    
    await asyncio.sleep(3)
    await start_night(context, game)

async def start_night(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Start night phase - YENİ ROLLER EKLENDİ"""
    logger.info(f"Grup {game.group_id}: Gece başlıyor")
    
    await clear_night_buttons(game)
    
    if game.phase != GamePhase.PLAYING:
        return
        
    game.phase = GamePhase.NIGHT
    game.night_actions = {"vampire": {}, "doctor": None, "kurt": None, "sapik": None, "yaramaz_kiz": None}
    
    vampires = [p for p in game.get_alive_players() if "Vampir" in p.role]
    doctor = next((p for p in game.get_alive_players() if "Doktor" in p.role), None)
    kurt = next((p for p in game.get_alive_players() if "Kurt" in p.role), None)
    sapik = next((p for p in game.get_alive_players() if p.role == ROLES["SAPIK"]), None)
    yaramaz_kiz = next((p for p in game.get_alive_players() if p.role == ROLES["YARAMAZ_KIZ"]), None)
    
    game.expected_voters = {p.user_id for p in vampires + 
                          ([doctor] if doctor else []) + 
                          ([kurt] if kurt else []) +
                          ([sapik] if sapik else []) +
                          ([yaramaz_kiz] if yaramaz_kiz else [])}
    
    for player in game.get_alive_players():
        if "Vampir" in player.role or "Doktor" in player.role or "Kurt" in player.role or player.role == ROLES["SAPIK"] or player.role == ROLES["YARAMAZ_KIZ"]:
            try:
                role_text = ""
                if "Vampir" in player.role:
                    role_text = "🌑 *GECE - VAMPİR SIRA*\n\n🩸 Kimi ısıracaksın?\n⏰ Süreniz: 60 saniye\n⚠️ Takım arkadaşınızı seçemezsiniz!"
                elif "Doktor" in player.role:
                    role_text = "💉 *GECE - DOKTOR SIRA*\n\n⛑️ Kimi koruyacaksın?\n⏰ Süreniz: 60 saniye\n⚠️ Takım arkadaşınızı seçemezsiniz!"
                elif "Kurt" in player.role:
                    role_text = "🐺 *GECE - ALFA KURT SIRA*\n\n⚔️ Kimi avlayacaksın?\n🎯 Sadece wampirleri öldürebilirsin!\n⏰ Süreniz: 60 saniye\n⚠️ Takım arkadaşını seçemezsin!"
                elif player.role == ROLES["SAPIK"]:
                    role_text = "😈 *GECE - SAPIK SIRA*\n\n🌙 Kimin koynuna gireceksin?\n💕 Romantik bir sürpriz yap!\n⏰ Süreniz: 60 saniye\n⚠️ Takım arkadaşını seçemezsin!"
                elif player.role == ROLES["YARAMAZ_KIZ"]:
                    role_text = "🔥 *GECE - YARAMAZ KIZ SIRA*\n\n🌙 Kimi ziyaret edeceksin?\n💃 Ateşli bir macera yaşa!\n⏰ Süreniz: 60 saniye\n⚠️ Takım arkadaşını seçemezsin!"
                
                message = await app.bot.send_message(
                    chat_id=player.user_id,
                    text=role_text,
                    reply_markup=build_player_buttons(game, group_id=game.group_id, phase="night"),
                    parse_mode="Markdown"
                )
                game.night_button_messages[player.user_id] = message.message_id
            except Exception as e:
                logger.error(f"Grup {game.group_id}: {player.username} gece buton hatası: {e}")
    
    special_roles_text = ""
    if sapik:
        special_roles_text += "😈 Sapık hazırlanıyor...\n"
    if yaramaz_kiz:
        special_roles_text += "🔥 Yaramaz Kız hazırlanıyor...\n"
        
    await safe_send_message(
        context, game.group_id,
        f"🌙 *GECE BAŞLADI!*\n\n🧛‍♂️ Vampirler avlanıyor...\n🩺 Doktor hazırlık yapıyor...\n🐺 Kurt ava çıkıyor...\n{special_roles_text}\n⏰ *Karar süresi: 60 saniye*"
    )
    
    if game._timer_task and not game._timer_task.done():
        game._timer_task.cancel()
    
    game._timer_task = asyncio.create_task(night_timer_60s(context, game))

async def night_timer_60s(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """60 saniye gece oylama timer'ı"""
    group_id = game.group_id
    
    for remaining in range(60, 0, -1):
        if game.phase != GamePhase.NIGHT:
            return
        await asyncio.sleep(1)
        
        if remaining == 30:
            await safe_send_message(context, group_id, "⚠️ *GECE UYARISI*\n\n⏳ 30 saniye kaldı!\n🧛‍♂️ Vampirler ve 🩺 Doktor hızlı karar versin!")
        elif remaining == 10:
            await safe_send_message(context, group_id, "🚨 *GECE SON 10 SANİYE!*\n\n⏰ Karar süreniz bitmek üzere!\nOy kullanmayanlar için otomatik devam edilecek!")
    
    if game.phase != GamePhase.NIGHT:
        return
        
    total_votes = len(game.night_actions["vampire"]) + (1 if game.night_actions["doctor"] else 0) + (1 if game.night_actions["kurt"] else 0) + (1 if game.night_actions["sapik"] else 0) + (1 if game.night_actions["yaramaz_kiz"] else 0)
    total_expected = len(game.expected_voters)
    
    await safe_send_message(
        context, group_id,
        f"🌅 *GECE SÜRESİ DOLDU!*\n\n📊 {total_votes}/{total_expected} kişi oy kullandı\n⚡ Kararlar işleniyor..."
    )
    
    await end_night(context, game)

async def end_night(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Gece sonu - ÖLÜM BİLDİRİMLİ"""
    group_id = game.group_id
    logger.info(f"Grup {group_id}: end_night çağrıldı!")
    
    if game.phase != GamePhase.NIGHT:
        return
    
    await clear_night_buttons(game)
    
    vampire_actions = len(game.night_actions["vampire"])
    doctor_action = bool(game.night_actions["doctor"])
    kurt_action = bool(game.night_actions["kurt"])
    sapik_action = bool(game.night_actions["sapik"])
    yaramaz_kiz_action = bool(game.night_actions["yaramaz_kiz"])
    
    night_summary = (
        f"🌅 *Gece Bitti*\n\n"
        f"🧛‍♂️ Vampirler: {'birini ısırdı' if vampire_actions > 0 else 'avlanmadı'}\n"
        f"🩺 Doktor: {'koruma yaptı' if doctor_action else 'koruma yapmadı'}\n"
        f"🐺 Alfa Kurt: {'avlandı' if kurt_action else 'avlanmadı'}\n"
        f"😈 Sapık: {'ziyaret etti' if sapik_action else 'ziyaret etmedi'}\n"
        f"🔥 Yaramaz Kız: {'ziyaret etti' if yaramaz_kiz_action else 'ziyaret etmedi'}"
    )
    
    await safe_send_message(context, group_id, night_summary)
    
    deaths = set()
    protected = game.night_actions["doctor"]
    kurt_target = game.night_actions["kurt"]
    
    # YENİ: Romantik ziyaretler
    if game.night_actions["sapik"]:
        await send_romantic_notification(context, game, next((p.user_id for p in game.players.values() if p.role == ROLES["SAPIK"]), None), game.night_actions["sapik"], "sapik")
    
    if game.night_actions["yaramaz_kiz"]:
        await send_romantic_notification(context, game, next((p.user_id for p in game.players.values() if p.role == ROLES["YARAMAZ_KIZ"]), None), game.night_actions["yaramaz_kiz"], "yaramaz_kiz")
    
    kurt_killed_vampire = False
    if kurt_target and kurt_target not in game.dead:
        kurt_target_player = game.players[kurt_target]
        if "Vampir" in kurt_target_player.role:
            if kurt_target != protected:
                deaths.add(kurt_target)
                kurt_killed_vampire = True
                await send_death_notification(context, game, kurt_target, "kurt_kill", f"🐺 Alfa Kurt seni avladı!")
    
    for vampire_id, target_id in game.night_actions["vampire"].items():
        if target_id == protected:
            await send_death_notification(context, game, target_id, "doctor_save", f"🩺 Doktor seni vampir saldırısından kurtardı!")
            vampire_name = game.players[vampire_id].username
            await send_death_notification(context, game, vampire_id, "protected", f"⛑️ Hedefin doktor tarafından korundu, ısıramadın!")
            continue
            
        if target_id in deaths:
            continue
            
        if target_id not in game.dead:
            deaths.add(target_id)
            vampire_name = game.players[vampire_id].username
            await send_death_notification(context, game, target_id, "vampire_night", f"🧛 {vampire_name} seni ısırdı!")
    
    if deaths:
        death_msg = "💀 *Gece Kurbanları:*\n"
        for death_id in deaths:
            game.kill_player(death_id)
            player_name = next(p.username for p in game.players.values() if p.user_id == death_id)
            death_msg += f"• {player_name} ({game.players[death_id].role})\n"
            await send_mention(context, group_id, death_id, "gece öldürüldü! 💀")
        
        await safe_send_message(context, group_id, death_msg)
    else:
        await safe_send_message(context, group_id, "🌙 Gece sakin geçti... Kimse ölmedi.")
    
    if check_win_condition(game):
        await end_game(context, game, is_night_end=True)
        return
    
    game.phase = GamePhase.PLAYING
    await asyncio.sleep(3)
    
    logger.info(f"Grup {group_id}: ☀️ Gündüz başlıyor...")
    await start_day(context, game)

async def start_day(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """90 saniye gündüz tartışma"""
    group_id = game.group_id
    logger.info(f"Grup {group_id}: Gündüz başlıyor - 90s tartışma")
    
    if game.phase != GamePhase.PLAYING:
        return
        
    game.phase = GamePhase.DAY
    game.votes = {}
    game.expected_voters = {p.user_id for p in game.get_alive_players()}
    
    await safe_send_message(
        context, group_id, 
        "☀️ *GÜNDÜZ BAŞLADI!*\n\n😱 Köylüler panik içinde uyandı!\n💀 Gece kurbanları arasında kayıplar var mı?\n🧛‍♂️ Vampirin kim olduğunu tartışın!\n\n⏰ *Tartışma süresi: 90 saniye*\n🗳️ Ardından oylama yapılacak!"
    )
    
    if game._timer_task and not game._timer_task.done():
        game._timer_task.cancel()
    
    game._timer_task = asyncio.create_task(discussion_timer(context, game))

async def discussion_timer(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """90 saniye tartışma timer'ı"""
    group_id = game.group_id
    
    notifications = {
        60: "⏳ *60 SANİYE KALDI!*\n\nTartışmalar kızışıyor... Şüphelerinizi paylaşın!",
        30: "⚠️ *30 SANİYE KALDI!*\n\nKarar verme zamanı yaklaşıyor! Kim şüpheli?",
        10: "🚨 *SON 10 SANİYE!*\n\nOylama başlıyor! Hızlıca son sözlerinizi söyleyin!"
    }
    
    for remaining in range(90, 0, -1):
        if game.phase != GamePhase.DAY:
            return
        await asyncio.sleep(1)
        
        if remaining in notifications:
            await safe_send_message(context, group_id, notifications[remaining])
    
    logger.info(f"Grup {group_id}: 💬 Tartışma aşaması bitti! BUTONLAR AÇILIYOR...")
    
    await safe_send_message(context, group_id, "⏰ *TARTIŞMA BİTTİ!*\n\n🗳️ Oylama başlıyor... Kimi linç edeceksiniz?")
    
    await start_voting(context, game)

async def start_voting(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """30 saniye gündüz oylama"""
    group_id = game.group_id
    logger.info(f"Grup {group_id}: 🗳️ 30 saniye oylama başlıyor!")
    
    if not game.expected_voters:
        await safe_send_message(context, group_id, "❌ Oy verecek canlı oyuncu yok! Gündüz iptal edildi.")
        await end_day(context, game)
        return
    
    vote_msg = (
        "🗳️ *OYLAMA BAŞLADI!*\n\n"
        "⚰️ Kimi linç edeceksiniz?\n"
        "👆 En çok oy alan idam edilecek!\n\n"
        "⏰ *Oylama süresi: 30 saniye*\n"
        "⚡ Oy kullanmayanlar otomatik geçilecek!"
    )
    
    markup = build_player_buttons(game, only_alive=True, group_id=group_id, phase="day")
    if not markup:
        await safe_send_message(context, group_id, "❌ Oy verecek canlı oyuncu yok! Gündüz iptal edildi.")
        await end_day(context, game)
        return
    
    sent_message = await context.bot.send_message(
        chat_id=group_id,
        text=vote_msg,
        reply_markup=markup,
        parse_mode="Markdown"
    )
    game.vote_message_id = sent_message.message_id
    
    logger.info(f"Grup {group_id}: 🗳️ Oylama butonları açıldı, ID: {game.vote_message_id}")
    
    if game._timer_task and not game._timer_task.done():
        game._timer_task.cancel()
    
    game._timer_task = asyncio.create_task(voting_timer(context, game))

async def voting_timer(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """30 saniye oylama timer'ı"""
    group_id = game.group_id
    
    await asyncio.sleep(15)
    if game.phase == GamePhase.DAY:
        total_voters = len(game.expected_voters)
        voted_count = len(game.votes)
        await safe_send_message(
            context, group_id,
            f"⚠️ *OYLAMA UYARISI*\n\n⏳ 15 saniye kaldı!\n📊 {voted_count}/{total_voters} kişi oy kullandı\n⚡ Kalanlar için otomatik devam edilecek!"
        )
    
    await asyncio.sleep(15)
    
    if game.phase == GamePhase.DAY:
        total_voters = len(game.expected_voters)
        voted_count = len(game.votes)
        logger.info(f"Grup {group_id}: 🗳️ 30 saniye DOLDU! {voted_count}/{total_voters} oy kullanıldı")
        
        await safe_send_message(
            context, group_id,
            f"⏰ *OYLAMA SÜRESİ DOLDU!*\n\n📊 {voted_count}/{total_voters} kişi oy kullandı\n⚖️ Oylar sayılıyor..."
        )
        
        await end_day(context, game)

async def end_day(context: ContextTypes.DEFAULT_TYPE, game: GameState):
    """Gündüz oylama sonuçları - LİNÇ BİLDİRİMLİ"""
    group_id = game.group_id
    logger.info(f"Grup {group_id}: ⚰️ Gündüz oylama sonuçları işleniyor...")
    
    if game.vote_message_id:
        try:
            await context.bot.edit_message_reply_markup(
                chat_id=group_id,
                message_id=game.vote_message_id,
                reply_markup=None
            )
            logger.info(f"Grup {group_id}: 🗑️ Gündüz oylama butonları KAPATILDI")
        except Exception as e:
            logger.error(f"Grup {group_id}: Gündüz buton kapatma hatası: {e}")
        finally:
            game.vote_message_id = None
    
    if game.phase != GamePhase.DAY:
        return
    
    total_voters = len(game.expected_voters)
    voted_count = len(game.votes)
    
    if not game.votes or voted_count == 0:
        await safe_send_message(
            context, group_id, 
            f"❌ *KİMSE OY KULLANMADI!*\n\n🤔 Köylüler kararsız kaldı ve kimse ölmedi!"
        )
        logger.info(f"Grup {group_id}: ⚖️ Hiç oy kullanılmadı - kimse ölmedi")
    else:
        vote_counts = {}
        voter_details = {}
        
        for voter_id, target_id in game.votes.items():
            vote_counts[target_id] = vote_counts.get(target_id, 0) + 1
            voter_name = game.players[voter_id].username
            target_name = game.players[target_id].username
            if target_id not in voter_details:
                voter_details[target_id] = []
            voter_details[target_id].append(voter_name)
        
        logger.info(f"Grup {group_id}: 🗳️ Oylama sonuçları: {vote_counts}")
        
        max_votes = max(vote_counts.values())
        candidates = [uid for uid, count in vote_counts.items() if count == max_votes]
        
        if len(candidates) > 1:
            candidate_names = [game.players[c].username for c in candidates]
            await safe_send_message(
                context, group_id,
                f"⚖️ *BERABERLİK!*\n\n🤔 *Köylüler kararsız kaldı ve kimse ölmedi!*"
            )
            logger.info(f"Grup {group_id}: ⚖️ Beraberlik - kimse ölmedi: {candidate_names}")
            
            distribution_msg = "🗳️ *Oy Dağılımı:*\n"
            for target_id, voters in voter_details.items():
                target_name = game.players[target_id].username
                distribution_msg += f"• {target_name}: {', '.join(voters)} ({len(voters)} oy)\n"
            
            distribution_msg += f"\n📈 Toplam {voted_count}/{total_voters} oy kullanıldı"
            await safe_send_message(context, group_id, distribution_msg, parse_mode="Markdown")
            
        else:
            target = candidates[0]
            target_player = game.players.get(target)
            
            if target_player:
                game.kill_player(target)
                
                voters_list = ", ".join(voter_details[target])
                await send_death_notification(context, game, target, "lynch_day", f"👥 Şu oyuncular sana oy verdi: {voters_list}")
                
                execution_msg = (
                    f"⚰️ *LİNÇ SONUCU*\n\n"
                    f"🎯 *İdam Edilen:* [{target_player.username}](tg://user?id={target})\n"
                    f"🎭 *Rolü:* {target_player.role}\n"
                    f"📊 *Oy Sayısı:* {max_votes}\n\n"
                )
                
                execution_msg += "🗳️ *Oy Dağılımı:*\n"
                for target_id, voters in voter_details.items():
                    target_name = game.players[target_id].username
                    execution_msg += f"• {target_name}: {', '.join(voters)} ({len(voters)} oy)\n"
                
                execution_msg += f"\n📈 Toplam {voted_count}/{total_voters} oy kullanıldı"
                
                await safe_send_message(context, group_id, execution_msg, parse_mode="Markdown")
                logger.info(f"Grup {group_id}: ⚰️ Linç: {target_player.username} ({target_player.role}) - {max_votes} oy")
                
                await send_mention(context, group_id, target, "linç edildi! 💀")
            else:
                await safe_send_message(context, group_id, "❌ Linç hatası!")
                logger.error(f"Grup {group_id}: ⚰️ Linç - Geçersiz hedef oyuncu")
    
    await asyncio.sleep(3)
    
    if check_win_condition(game):
        await end_game(context, game)
        return
    
    game.phase = GamePhase.PLAYING
    await asyncio.sleep(3)
    
    await safe_send_message(
        context, group_id, 
        "🌙 *YENİ GECE BAŞLIYOR...*\n\n"
        "👻 Kötü rüyalar görecek olanlar var!\n"
        "⏰ Gecenin karanlığında av başlıyor...\n"
        "⚡ Otomatik devam ediliyor!"
    )
    
    await asyncio.sleep(3)
    await start_night(context, game)

def check_win_condition(game: GameState) -> bool:
    """Check if game should end"""
    alive_players = game.get_alive_players()
    alive_vampires = sum(1 for p in alive_players if "Vampir" in p.role)
    alive_non_vampires = len(alive_players) - alive_vampires
    
    if alive_vampires >= alive_non_vampires and alive_vampires > 0:
        return True
    if alive_vampires == 0:
        return True
    return False

async def end_game(context: ContextTypes.DEFAULT_TYPE, game: GameState, is_night_end: bool = False):
    """Oyun sonu - TÜM OYUNCULARA BİLDİRİM"""
    group_id = game.group_id
    game.set_active(False)
    
    alive_vampires = any(p.alive and "Vampir" in p.role for p in game.players.values())
    winner = "🧛‍♂️ Vampirler" if alive_vampires else "👨‍🌾 Köylüler"
    
    # YENİ: Tüm oyunculara oyun sonu bildirimi
    for player in game.players.values():
        status = "🎉 KAZANDIN!" if (
            (alive_vampires and "Vampir" in player.role and player.alive) or
            (not alive_vampires and "Vampir" not in player.role and player.alive)
        ) else "😞 KAYBETTİN!"
        
        end_message = (
            f"🏆 *OYUN BİTTİ!*\n\n"
            f"{status}\n"
            f"🎭 Rolün: {player.role}\n"
            f"❤️ Durum: {'Hayatta' if player.alive else 'Ölü'}\n"
            f"🏅 Kazanan: {winner}\n\n"
            f"🔄 Yeni oyun için /wstart yazın!"
        )
        
        await safe_send_pm(player.user_id, end_message)
    
    image_url = IMAGES["VAMPIR_WIN"] if alive_vampires else IMAGES["KOYLU_WIN"]
    results_text = f"🏆 *{winner} Kazandı!*\n\n📊 *Son Durum:*\n"
    
    for player in game.players.values():
        status = "💀 Öldü" if not player.alive else "❤️ Hayatta"
        results_text += f"• {player.username}: {player.role} - {status}\n"
    
    await safe_send_photo(context, group_id, image_url, results_text, parse_mode="Markdown")
    
    logger.info(f"Grup {group_id}: 🏆 Oyun bitti! Kazanan: {winner}")
    
    await asyncio.sleep(5)
    game.reset()
    
    await safe_send_message(context, group_id, "🔄 Oyun bitti! Yeni oyun için /wstart kullanın.")

# === CALLBACK HANDLER ===

async def handle_join_button(query, context: ContextTypes.DEFAULT_TYPE):
    """Oyuna katıl butonu işlemi"""
    user = query.from_user
    group_id = query.message.chat_id
    
    game = get_game(group_id)
    
    if not game.is_active() or game.phase != GamePhase.LOBBY:
        await query.answer("❌ Bu oyun artık aktif değil!", show_alert=True)
        return
    
    if user.id in game.players:
        await query.answer("❌ Zaten bu oyundasınız!", show_alert=True)
        return
    
    try:
        test_msg = await context.bot.send_message(
            chat_id=user.id,
            text="🤖 *Vampir Köylü Botu*\n\n🎮 Oyuna katılmak için aşağıdaki butona tıklayın!",
            reply_markup=InlineKeyboardMarkup([[
                InlineKeyboardButton("🎮 Oyuna Katıl", callback_data=f"pm_join_{group_id}")
            ]]),
            parse_mode="Markdown"
        )
        
        await direct_join_game(user, game, context, query)
        
    except Exception as e:
        await query.answer("🤖 Botla iletişim kurmanız gerekiyor! Özelden /start yazın.", show_alert=True)
        
        help_text = (
            f"🎮 Merhaba {user.first_name}!\n\n"
            f"Oyuna katılmak için:\n"
            f"1. 🤖 @Wwampir_bot'a tıklayın\n"
            f"2. Özelden 'Merhaba' veya /start yazın\n"
            f"3. Buraya dönüp butona tekrar tıklayın"
        )
        
        await query.message.reply_text(
            help_text,
            reply_markup=InlineKeyboardMarkup([[
                InlineKeyboardButton("🤖 Bota Git", url="https://t.me/Wwampir_bot")
            ]]),
            parse_mode="Markdown"
        )

async def direct_join_game(user, game: GameState, context: ContextTypes.DEFAULT_TYPE, query=None):
    """Direkt oyuna katıl"""
    game.add_player(user.id, user.first_name or user.username or "Bilinmeyen")
    
    if query:
        await query.answer("🎉 Oyuna katıldınız!")
    
    await send_mention(context, game.group_id, user.id, "oyuna katıldı! 🎉")
    await update_join_message(context, game)
    
    player_count = len(game.players)
    logger.info(f"👥 Grup {game.group_id}: Butonla katılım: {player_count} kişi")
    
    if player_count == 5:
        if game._join_timer_task and not game._join_timer_task.done():
            game._join_timer_task.cancel()
        
        game.join_time_left = 60
        game._join_timer_task = asyncio.create_task(join_countdown(context, game))
        
        await safe_send_message(context, game.group_id, "🎉 5 kişi tamamlandı!\n⏳ 1 dakika içinde başka oyuncu katılmazsa oyun başlayacak.")
    elif player_count > 5 and game._join_timer_task and not game._join_timer_task.done():
        game.join_time_left = 60
        await safe_send_message(context, game.group_id, f"➕ Yeni oyuncu! Süre 60 saniyeye sıfırlandı.\n👥 Toplam: {player_count} oyuncu")

async def handle_pm_join_button(query, context: ContextTypes.DEFAULT_TYPE):
    """PM'den katılma butonu"""
    user = query.from_user
    parts = query.data.split("_")
    
    if len(parts) < 3:
        await query.answer("❌ Geçersiz buton!", show_alert=True)
        return
    
    try:
        group_id = int(parts[2])
    except ValueError:
        await query.answer("❌ Geçersiz buton!", show_alert=True)
        return
    
    if group_id not in games:
        await query.answer("❌ Bu oyun artık aktif değil!", show_alert=True)
        return
    
    game = games[group_id]
    
    if not game.is_active() or game.phase != GamePhase.LOBBY:
        await query.answer("❌ Bu oyun artık aktif değil!", show_alert=True)
        return
    
    if user.id in game.players:
        await query.answer("❌ Zaten bu oyundasınız!", show_alert=True)
        return
    
    await direct_join_game(user, game, context, query)

async def handle_night_action(query, user_id, target_id, context, game):
    """Gece aksiyonları - YENİ ROLLER EKLENDİ"""
    group_id = game.group_id
    player = game.players[user_id]
    target_player = game.players[target_id]
    
    if player.role == target_player.role and player.role != ROLES["KOYLU"]:
        await query.answer("⚠️ Takım arkadaşına aksiyon uygulayamazsın!", show_alert=True)
        return
    
    action_msg = ""
    if "Vampir" in player.role:
        if user_id in game.night_actions["vampire"]:
            await query.answer("⚠️ Zaten oy kullandın!", show_alert=True)
            return
        game.night_actions["vampire"][user_id] = target_id
        action_msg = f"🩸 {target_player.username} ısırıldı!"
        await safe_send_message(context, group_id, "🧛‍♂️ Bir vampir avına çıktı!")
    elif "Doktor" in player.role:
        if game.night_actions["doctor"] is not None:
            await query.answer("⚠️ Zaten koruma seçtin!", show_alert=True)
            return
        game.night_actions["doctor"] = target_id
        action_msg = f"⛑️ {target_player.username} korundu!"
        await safe_send_message(context, group_id, "🩺 Doktor şifa dağıtıyor!")
    elif "Kurt" in player.role:
        if game.night_actions["kurt"] is not None:
            await query.answer("⚠️ Zaten av seçtin!", show_alert=True)
            return
        game.night_actions["kurt"] = target_id
        action_msg = f"🐺 {target_player.username} avlandı!"
        await safe_send_message(context, group_id, "🐺 Alfa Kurt ava çıktı!")
    elif player.role == ROLES["SAPIK"]:
        if game.night_actions["sapik"] is not None:
            await query.answer("⚠️ Zaten birinin koynuna girdin!", show_alert=True)
            return
        game.night_actions["sapik"] = target_id
        action_msg = f"😈 {target_player.username}'in koynuna girdin!"
        await send_romantic_notification(context, game, user_id, target_id, "sapik")
    elif player.role == ROLES["YARAMAZ_KIZ"]:
        if game.night_actions["yaramaz_kiz"] is not None:
            await query.answer("⚠️ Zaten birini ziyaret ettin!", show_alert=True)
            return
        game.night_actions["yaramaz_kiz"] = target_id
        action_msg = f"🔥 {target_player.username}'i ziyaret ettin!"
        await send_romantic_notification(context, game, user_id, target_id, "yaramaz_kiz")
    else:
        await query.answer("❌ Bu aşamada oy kullanamazsınız!", show_alert=True)
        return
    
    geri_bildirim_msg = ""
    if "Vampir" in player.role:
        geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli oyuncuyu ısırdın!\n\n🩸 Bu kişi doktor tarafından korunmazsa ölecek."
    elif "Doktor" in player.role:
        geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli oyuncuyu koruyorsun!\n\n⛑️ Bu kişi vampir saldırısından kurtulacak."
    elif "Kurt" in player.role:
        if "Vampir" in target_player.role:
            geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli VAMPİR'i avladın!\n\n🐺 Bu wampir ölecek!"
        else:
            geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli oyuncuyu avlamaya çalıştın!\n\n⚠️ Bu kişi wampir değil, zarar veremezsin."
    elif player.role == ROLES["SAPIK"]:
        geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli oyuncunun koynuna girdin!\n\n😈 Bu kişiye romantik bir sürpriz yaptın!"
    elif player.role == ROLES["YARAMAZ_KIZ"]:
        geri_bildirim_msg = f"🎯 *Gece Kararın:* {target_player.username} isimli oyuncuyu ziyaret ettin!\n\n🔥 Bu kişiye ateşli bir macera yaşattın!"
    
    await safe_send_pm(user_id, geri_bildirim_msg)
    await query.answer(action_msg)

async def handle_day_vote(query, user_id, target_id, context, game):
    """Handle day voting"""
    group_id = game.group_id
    player = game.players[user_id]
    target_player = game.players[target_id]
    
    if user_id in game.votes:
        await query.answer("⚠️ Zaten oy kullandınız!", show_alert=True)
        return
    
    game.votes[user_id] = target_id
    action_msg = f"🗳️ {target_player.username} için oy verdiniz!"
    
    await query.answer(action_msg)
    logger.info(f"Grup {group_id}: 🗳️ {player.username} -> {target_player.username} oy verdi")
    
    vote_announcement = (
        f"🗳️ [{player.username}](tg://user?id={user_id}), "
        f"[{target_player.username}](tg://user?id={target_id})'yi linç etmeyi seçti!"
    )
    
    await safe_send_message(context, group_id, vote_announcement, parse_mode="Markdown")
    
    if len(game.votes) >= len(game.expected_voters):
        logger.info(f"Grup {group_id}: 🗳️ Herkes oy kullandı! Oylama erken bitiyor...")
        if game._timer_task and not game._timer_task.done():
            game._timer_task.cancel()
        await end_day(context, game)

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle inline button callbacks"""
    query = update.callback_query
    await query.answer()
    
    if query.data.startswith("pm_join_"):
        await handle_pm_join_button(query, context)
        return
    
    if query.data == "join_game":
        await handle_join_button(query, context)
        return
    
    if query.data in ["help_rules", "help_commands"]:
        if query.data == "help_rules":
            rules_text = (
                "🧛‍♂️ *Vampir Köylü - Hızlı Rehber*\n\n"
                "🎯 *Amaç:*\n"
                "• 🧛 Vampirler: Tüm köylüleri öldür\n"
                "• 👨‍🌾 Köylüler: Tüm vampirleri bul\n\n"
                "🔄 *Oyun Döngüsü:*\n"
                "🌙 *Gece* → Vampirler saldırır, Doktor korur\n"
                "☀️ *Gündüz* → Tartışma + Oylama\n\n"
                "🏆 *Kazanma:*\n"
                "• Vampirler: Canlı vampir ≥ canlı köylü\n"
                "• Köylüler: Tüm vampirler ölü\n\n"
                "📚 *Detaylı rehber için:* `/wbilgi`"
            )
            await query.message.edit_text(rules_text, parse_mode="Markdown")
        else:
            help_text = (
                "🧛‍♂️ *Vampir Köylü - Komutlar*\n\n"
                "🎮 **Oyun Yönetimi:**\n"
                "• `/wstart` - Oyunu başlat\n"
                "• `/wjoin` - Oyuna katıl\n"
                "• `/wson` - Oyunu iptal et\n"
                "• `/wextend <dakika>` - Extra süre ekle\n\n"
                "📋 **Bilgi:**\n"
                "• `/wbilgi` - Detaylı oyun rehberi\n"
                "• `/whelp` - Yardım mesajı"
            )
            await query.message.edit_text(help_text, parse_mode="Markdown")
        return
    
    if not query.data or not query.data.startswith("target_"):
        return
    
    parts = query.data.split("_")
    if len(parts) < 4:
        await query.answer("❌ Geçersiz buton!", show_alert=True)
        return

    try:
        group_id = int(parts[1])
        target_id = int(parts[2])
        button_phase = parts[3]
    except (ValueError, IndexError):
        await query.answer("❌ Geçersiz buton!", show_alert=True)
        return
    
    if group_id not in games:
        await query.answer("❌ Bu grupta oyun yok!", show_alert=True)
        return
    
    game = games[group_id]
    user_id = query.from_user.id
    
    if not game.is_active():
        await query.answer("❌ Bu grupta aktif oyun yok!", show_alert=True)
        return
    
    if user_id not in game.players:
        await query.answer("❌ Bu oyunda değilsiniz!", show_alert=True)
        return
    
    player = game.players[user_id]
    if not player.alive:
        await query.answer("💀 Ölüler oy kullanamaz!", show_alert=True)
        return
    
    target_player = game.players.get(target_id)
    if not target_player or not target_player.alive:
        await query.answer("❌ Ölü birine oy veremezsiniz!", show_alert=True)
        return
    
    current_phase = "night" if game.phase == GamePhase.NIGHT else "day" if game.phase == GamePhase.DAY else "other"
    
    if button_phase != current_phase:
        await query.answer("⏰ Bu butonun süresi doldu! Artık kullanılamaz.", show_alert=True)
        return
    
    if game.phase == GamePhase.NIGHT:
        await handle_night_action(query, user_id, target_id, context, game)
    elif game.phase == GamePhase.DAY:
        await handle_day_vote(query, user_id, target_id, context, game)
    else:
        await query.answer("⚠️ Şu anda oy kullanılamaz.", show_alert=True)

# === MAIN APPLICATION ===

def main():
    """Basit main fonksiyonu"""
    global app
    
    logger.info("🧛‍♂️ Vampir Köylü Botu başlatılıyor...")
    
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("wstart", wstart))
    app.add_handler(CommandHandler("wjoin", wjoin))
    app.add_handler(CommandHandler("wson", wson))
    app.add_handler(CommandHandler("whelp", whelp))
    app.add_handler(CommandHandler("wbilgi", wbilgi))
    app.add_handler(CommandHandler("wnasiloynanir", wnasıloynanır))
    app.add_handler(CommandHandler("wextend", wextend))
    app.add_handler(CallbackQueryHandler(button_handler))
    
    async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
        logger.error(f"Error: {context.error}")
    
    app.add_error_handler(error_handler)
    
    logger.info("🧛‍♂️ Vampir Köylü Botu aktif!")
    print("🧛‍♂️ Vampir Köylü Botu Aktif!")
    print("Grup chat'inde /wstart yazın!")
    
    try:
        app.run_polling(drop_pending_updates=True)
    except KeyboardInterrupt:
        logger.info("Bot kapatılıyor...")
    except Exception as e:
        logger.error(f"Bot hatası: {e}")

if __name__ == "__main__":
    main()
