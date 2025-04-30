# twitch-music-obs-plugin
teste
## Estrutura inicial do projeto

# main.py
from twitchio.ext import commands
import asyncio
import subprocess
from queue import Queue

music_queue = Queue()
currently_playing = False

class Bot(commands.Bot):
    def __init__(self):
        super().__init__(
            token='SEU_TOKEN_OAUTH',
            prefix='!',
            initial_channels=['SEU_CANAL']
        )

    async def event_ready(self):
        print(f'Bot conectado como {self.nick}')

    @commands.command(name='play')
    async def play(self, ctx):
        query = ctx.message.content[6:].strip()
        if not query:
            await ctx.send('Uso: !play <link ou nome da música>')
            return

        music_queue.put(query)
        await ctx.send(f'🎵 "{query}" adicionada à fila!')

        if not currently_playing:
            asyncio.create_task(play_next(ctx))

    @commands.command(name='skip')
    async def skip(self, ctx):
        subprocess.call(['pkill', '-f', 'ffplay'])  # Termina o processo do ffplay
        await ctx.send('⏭️ Música pulada!')

async def play_next(ctx):
    global currently_playing

    while not music_queue.empty():
        currently_playing = True
        query = music_queue.get()
        await ctx.send(f'Tocando agora: {query}')

        # Baixar áudio com yt-dlp
        subprocess.call([
            'yt-dlp',
            '--extract-audio',
            '--audio-format', 'mp3',
            '-o', 'current.mp3',
            query
        ])

        # Tocar com ffmpeg/ffplay
        process = subprocess.Popen([
            'ffplay', '-nodisp', '-autoexit', 'current.mp3'
        ])
        process.wait()

    currently_playing = False

if __name__ == '__main__':
    bot = Bot()
    bot.run()
