# twitch-music-obs-plugin

## Estrutura inicial do projeto

# main.py
from twitchio.ext import commands
import asyncio
import subprocess
from queue import Queue

music_queue = Queue()
currently_playing = False
queued_songs = set()

class MusicBot(commands.Bot):
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

        if query in queued_songs:
            await ctx.send(f'⛔ "{query}" já está na fila!')
            return

        music_queue.put(query)
        queued_songs.add(query)
        await ctx.send(f'🎵 "{query}" adicionada à fila!')

        global currently_playing
        if not currently_playing:
            currently_playing = True
            asyncio.create_task(play_next(ctx))

    @commands.command(name='skip')
    async def skip(self, ctx):
        subprocess.call(['pkill', '-f', 'ffplay'])  # Termina o processo do ffplay
        await ctx.send('⏩️ Música pulada!')

    @commands.command(name='fila')
    async def fila(self, ctx):
        if music_queue.empty():
            await ctx.send('A fila está vazia!')
        else:
            fila_list = list(queued_songs)
            await ctx.send('Fila atual: ' + ' | '.join(fila_list))

async def play_next(ctx):
    global currently_playing

    while not music_queue.empty():
        query = music_queue.get()
        queued_songs.discard(query)
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

# Execução segura do bot sem uso de asyncio.run()
async def main():
    bot = MusicBot()
    await bot.start()

if __name__ == '__main__':
    try:
        asyncio.get_event_loop().run_until_complete(main())
    except KeyboardInterrupt:
        print("Bot finalizado.")
