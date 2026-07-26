import discord
from discord.ext import commands
from discord import app_commands
import asyncio
import random
import re

# ==========================
# CONFIGURAÇÃO
# ==========================

TOKEN = "MTUyOTMzMjAxMDYzNDU3NTg3Mg.Gldc_7.pZ0hVzJWvRX7RUCeRJH83lux-3G3YMjWuo1aT4"
GUILD_ID = 1527055736561995806

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(
    command_prefix="!",
    intents=intents
)

# Guarda os participantes dos sorteios
participantes = {}

# ==========================
# BOTÃO PARTICIPAR
# ==========================

class ParticiparView(discord.ui.View):

    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(
        label="Participar",
        emoji="🎉",
        style=discord.ButtonStyle.green,
        custom_id="participar_sorteio"
    )
    async def participar(
        self,
        interaction: discord.Interaction,
        button: discord.ui.Button
    ):

        lista = participantes.setdefault(
            interaction.message.id,
            set()
        )

        if interaction.user.id in lista:
            await interaction.response.send_message(
                "❌ Você já está participando deste sorteio.",
                ephemeral=True
            )
            return

        lista.add(interaction.user.id)

        await interaction.response.send_message(
            "✅ Você entrou no sorteio!",
            ephemeral=True
        )

# ==========================
# EVENTO READY
# ==========================

@bot.event
async def on_ready():

    guild = discord.Object(id=GUILD_ID)

    bot.add_view(ParticiparView())

    sincronizados = await bot.tree.sync(guild=guild)

    print("=" * 40)
    print(f"{bot.user} está online!")
    print(f"{len(sincronizados)} comandos sincronizados.")
    print("=" * 40)

# ==========================
# /AJUDA
# ==========================

@bot.tree.command(
    name="ajuda",
    description="Mostra todos os comandos.",
    guild=discord.Object(id=GUILD_ID)
)
async def ajuda(interaction: discord.Interaction):

    embed = discord.Embed(
        title="📖 Painel de Ajuda",
        color=discord.Color.blue()
    )

    embed.add_field(
        name="Comandos",
        value=(
            "📢 /anuncios\n"
            "🎉 /sorteio\n"
            "🏰 /clan\n"
            "🔇 /mute\n"
            "📖 /ajuda"
        ),
        inline=False
    )

    await interaction.response.send_message(embed=embed)

# ==========================
# /ANUNCIOS
# ==========================

@app_commands.describe(
    mensagem="Mensagem do anúncio"
)

@bot.tree.command(
    name="anuncios",
    description="Enviar anúncio",
    guild=discord.Object(id=GUILD_ID)
)
async def anuncios(
    interaction: discord.Interaction,
    mensagem: str
):

    if not interaction.user.guild_permissions.administrator:

        await interaction.response.send_message(
            "❌ Apenas administradores podem usar este comando.",
            ephemeral=True
        )
        return

    await interaction.response.send_message(
        "✅ Anúncio enviado.",
        ephemeral=True
    )

    await interaction.channel.send(
        f"@everyone @here\n\n📢 **ANÚNCIO**\n\n{mensagem}",
        allowed_mentions=discord.AllowedMentions(
            everyone=True
        )
    )
    # ==========================
# MODAL DO SORTEIO
# ==========================

class SorteioModal(discord.ui.Modal, title="Criar Sorteio"):

    premio = discord.ui.TextInput(
        label="🎁 Prêmio",
        required=True
    )

    minutos = discord.ui.TextInput(
        label="⏰ Duração (minutos)",
        placeholder="Ex: 10",
        required=True
    )

    descricao = discord.ui.TextInput(
        label="📝 Informações",
        style=discord.TextStyle.paragraph,
        required=False
    )

    async def on_submit(self, interaction: discord.Interaction):

        try:
            tempo = int(self.minutos.value)

            if tempo <= 0:
                raise ValueError

        except ValueError:
            await interaction.response.send_message(
                "❌ Informe um número válido de minutos.",
                ephemeral=True
            )
            return

        embed = discord.Embed(
            title="🎉 NOVO SORTEIO",
            description="Clique no botão abaixo para participar!",
            color=discord.Color.gold()
        )

        embed.add_field(
            name="🎁 Prêmio",
            value=self.premio.value,
            inline=False
        )

        embed.add_field(
            name="⏰ Duração",
            value=f"{tempo} minuto(s)",
            inline=False
        )

        if self.descricao.value:

            embed.add_field(
                name="📝 Informações",
                value=self.descricao.value,
                inline=False
            )

        embed.set_footer(text="Boa sorte!")

        mensagem = await interaction.channel.send(
            embed=embed,
            view=ParticiparView()
        )

        participantes[mensagem.id] = set()

        await interaction.response.send_message(
            "✅ Sorteio criado com sucesso!",
            ephemeral=True
        )

        # Aguarda o término
        await asyncio.sleep(tempo * 60)

        inscritos = list(
            participantes.get(
                mensagem.id,
                set()
            )
        )

        # Sem participantes
        if len(inscritos) == 0:

            encerrado = discord.Embed(
                title="❌ Sorteio Encerrado",
                description="Nenhum participante entrou no sorteio.",
                color=discord.Color.red()
            )

            await interaction.channel.send(embed=encerrado)

            participantes.pop(mensagem.id, None)
            return

        # Escolhe vencedor
        vencedor_id = random.choice(inscritos)

        vencedor = interaction.guild.get_member(vencedor_id)

        resultado = discord.Embed(
            title="🏆 SORTEIO ENCERRADO",
            color=discord.Color.green()
        )

        resultado.add_field(
            name="🎁 Prêmio",
            value=self.premio.value,
            inline=False
        )

        if vencedor:

            resultado.add_field(
                name="🥳 Vencedor",
                value=vencedor.mention,
                inline=False
            )

            await interaction.channel.send(
                content=f"🎉 Parabéns {vencedor.mention}!",
                embed=resultado
            )

        else:

            resultado.add_field(
                name="🥳 Vencedor",
                value=f"<@{vencedor_id}>",
                inline=False
            )

            await interaction.channel.send(embed=resultado)

        participantes.pop(mensagem.id, None)


# ==========================
# /SORTEIO
# ==========================

@bot.tree.command(
    name="sorteio",
    description="Criar um sorteio",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.checks.has_permissions(administrator=True)
async def sorteio(interaction: discord.Interaction):

    await interaction.response.send_modal(
        SorteioModal()
    )


@sorteio.error
async def sorteio_error(interaction: discord.Interaction, error):

    if isinstance(error, app_commands.MissingPermissions):

        await interaction.response.send_message(
            "❌ Apenas administradores podem criar sorteios.",
            ephemeral=True
        )
 # ==========================
# /CLAN
# ==========================

class ClanSelect(discord.ui.Select):

    def __init__(self):

        options = [
            discord.SelectOption(
                label="Sem Clan",
                value="Sem Clan",
                emoji="❌"
            ),
            discord.SelectOption(
                label="0102",
                value="0102",
                emoji="⚔️"
            ),
            discord.SelectOption(
                label="JKR",
                value="JKR",
                emoji="👑"
            )
        ]

        super().__init__(
            placeholder="Escolha seu clan...",
            min_values=1,
            max_values=1,
            options=options
        )

    async def callback(self, interaction: discord.Interaction):

        membro = interaction.user
        guild = interaction.guild

        cargo_sem_clan = discord.utils.get(
            guild.roles,
            name="Sem Clan"
        )

        cargo_0102 = discord.utils.get(
            guild.roles,
            name="0102"
        )

        cargo_jkr = discord.utils.get(
            guild.roles,
            name="[🎮 ] Membro"
        )

        # Remove os cargos antigos
        for cargo in [cargo_sem_clan, cargo_0102, cargo_jkr]:
            if cargo and cargo in membro.roles:
                await membro.remove_roles(cargo)

        escolha = self.values[0]

        if escolha == "Sem Clan" and cargo_sem_clan:
            await membro.add_roles(cargo_sem_clan)

        elif escolha == "0102" and cargo_0102:
            await membro.add_roles(cargo_0102)

        elif escolha == "JKR" and cargo_jkr:
            await membro.add_roles(cargo_jkr)

        await interaction.response.send_message(
            f"✅ Seu clan agora é **{escolha}**.",
            ephemeral=True
        )


class ClanView(discord.ui.View):

    def __init__(self):
        super().__init__(timeout=None)

        self.add_item(
            ClanSelect()
        )


@bot.tree.command(
    name="clan",
    description="Escolher seu clan",
    guild=discord.Object(id=GUILD_ID)
)
async def clan(interaction: discord.Interaction):

    embed = discord.Embed(
        title="🏰 Escolha seu Clan",
        description=(
            "Selecione abaixo o clan ao qual você pertence.\n\n"
            "O cargo será alterado automaticamente."
        ),
        color=discord.Color.blue()
    )

    embed.set_footer(
        text="Você pode alterar seu clan quando quiser."
    )

    await interaction.response.send_message(
        embed=embed,
        view=ClanView(),
        ephemeral=True
    )
 # ==========================
# /MUTE
# ==========================

def converter_tempo(tempo: str):

    match = re.fullmatch(r"(\d+)([smhd])", tempo.lower())

    if not match:
        return None

    valor = int(match.group(1))
    unidade = match.group(2)

    if unidade == "s":
        return valor

    elif unidade == "m":
        return valor * 60

    elif unidade == "h":
        return valor * 3600

    elif unidade == "d":
        return valor * 86400

    return None


@bot.tree.command(
    name="mute",
    description="Silencia um membro temporariamente.",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    membro="Membro que será silenciado",
    tempo="Ex: 30m, 2h ou 1d"
)
@app_commands.checks.has_permissions(
    moderate_members=True
)
async def mute(
    interaction: discord.Interaction,
    membro: discord.Member,
    tempo: str
):

    segundos = converter_tempo(tempo)

    if segundos is None:

        await interaction.response.send_message(
            "❌ Tempo inválido.\nUse exemplos como: **30m**, **2h** ou **1d**.",
            ephemeral=True
        )
        return

    cargo = discord.utils.get(
        interaction.guild.roles,
        name="Mutado"
    )

    # Cria o cargo automaticamente
    if cargo is None:

        cargo = await interaction.guild.create_role(
            name="Mutado",
            colour=discord.Color.dark_grey(),
            reason="Cargo criado automaticamente."
        )

        for canal in interaction.guild.channels:

            try:
                await canal.set_permissions(
                    cargo,
                    send_messages=False,
                    speak=False,
                    add_reactions=False
                )
            except Exception:
                pass

    await membro.add_roles(cargo)

    embed = discord.Embed(
        title="🔇 Usuário Mutado",
        color=discord.Color.red()
    )

    embed.add_field(
        name="👤 Usuário",
        value=membro.mention,
        inline=False
    )

    embed.add_field(
        name="⏳ Duração",
        value=tempo,
        inline=False
    )

    embed.set_footer(
        text="O mute será removido automaticamente."
    )

    await interaction.response.send_message(
        embed=embed
    )

    await asyncio.sleep(segundos)

    if cargo in membro.roles:

        await membro.remove_roles(cargo)

        await interaction.channel.send(
            f"🔊 {membro.mention} foi desmutado automaticamente."
        )


# ==========================
# ERRO DO /MUTE
# ==========================

@mute.error
async def mute_error(interaction, error):

    if isinstance(error, app_commands.MissingPermissions):

        await interaction.response.send_message(
            "❌ Você não possui permissão para usar este comando.",
            ephemeral=True
        )

    elif isinstance(error, app_commands.CommandInvokeError):

        await interaction.response.send_message(
            "❌ Ocorreu um erro ao executar este comando.",
            ephemeral=True
        )

# ==========================
# SISTEMA DE TICKETS
# ==========================

class TicketSelect(discord.ui.Select):

    def __init__(self):

        options = [
            discord.SelectOption(
                label="🎮 Testes",
                value="testes",
                description="Abrir ticket para testes"
            ),
            discord.SelectOption(
                label="❓ Dúvidas",
                value="duvidas",
                description="Abrir ticket para dúvidas"
            ),
            discord.SelectOption(
                label="🤝 Parceria",
                value="parceria",
                description="Abrir ticket para parceria"
            )
        ]

        super().__init__(
            custom_id="ticket_select_menu",  # Obrigatório
            placeholder="Selecione o motivo do ticket...",
            min_values=1,
            max_values=1,
            options=options
        )

    async def callback(self, interaction: discord.Interaction):

        categoria = discord.utils.get(interaction.guild.categories, name="Tickets")

        if categoria is None:
            categoria = await interaction.guild.create_category("Tickets")

        nome = f"{self.values[0]}-{interaction.user.name}".lower().replace(" ", "-")

        # Não permite dois tickets do mesmo usuário
        for canal in categoria.text_channels:
            if canal.name.endswith(interaction.user.name.lower().replace(" ", "-")):
                await interaction.response.send_message(
                    "❌ Você já possui um ticket aberto.",
                    ephemeral=True
                )
                return

        overwrites = {
            interaction.guild.default_role: discord.PermissionOverwrite(
                view_channel=False
            ),

            interaction.user: discord.PermissionOverwrite(
                view_channel=True,
                send_messages=True,
                read_message_history=True,
                attach_files=True
            ),

            interaction.guild.me: discord.PermissionOverwrite(
                view_channel=True,
                send_messages=True
            )
        }

        cargos = [
            "[⚜️] Sub Dono",
            "[⚖️] Conselho",
            "[🏆 ] Tester Superior",
            "[🛠️] Dev"
        ]

        for nome_cargo in cargos:
            cargo = discord.utils.get(interaction.guild.roles, name=nome_cargo)

            if cargo:
                overwrites[cargo] = discord.PermissionOverwrite(
                    view_channel=True,
                    send_messages=True,
                    read_message_history=True,
                    manage_messages=True
                )

        canal = await interaction.guild.create_text_channel(
            name=nome,
            category=categoria,
            overwrites=overwrites
        )

        embed = discord.Embed(
            title="🎫 Ticket Aberto",
            description=(
                f"Categoria: **{self.values[0].capitalize()}**\n\n"
                "Explique seu problema e aguarde a equipe."
            ),
            color=discord.Color.green()
        )

        await canal.send(
            content=interaction.user.mention,
            embed=embed
        )

        await interaction.response.send_message(
            f"✅ Ticket criado: {canal.mention}",
            ephemeral=True
        )


class TicketView(discord.ui.View):

    def __init__(self):
        super().__init__(timeout=None)
        self.add_item(TicketSelect())


# ==========================
# /TICKET
# ==========================

@bot.tree.command(
    name="ticket",
    description="Envia o painel de tickets.",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.checks.has_permissions(administrator=True)
async def ticket(interaction: discord.Interaction):

    embed = discord.Embed(
        title="🎫 Central de Atendimento",
        description=(
            "Escolha uma opção abaixo para abrir um ticket.\n\n"
            "🎮 **Testes** Feito para você. Se você quer entrar no clã JKR, solicite um teste.\n"
            "   "
            "❓ **Dúvidas** Para você tirar suas dúvidas com um administrador.\n"
            "   "
            "🤝 **Parceria** Para você realizar uma parceria com o clã JKR.\n"
        ),
        color=discord.Color.blurple()
    )

    embed.set_footer(text="Selecione uma categoria no menu abaixo.")

    await interaction.response.send_message(
        embed=embed,
        view=TicketView()
    )


class TicketView(discord.ui.View):

    def __init__(self):
        super().__init__(timeout=None)
        self.add_item(TicketSelect())



        # ==========================
# BOTÃO FECHAR TICKET
# ==========================

class FecharTicketView(discord.ui.View):

    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(
        label="Fechar Ticket",
        emoji="🔒",
        style=discord.ButtonStyle.red,
        custom_id="fechar_ticket"
    )
    async def fechar(
        self,
        interaction: discord.Interaction,
        button: discord.ui.Button
    ):

        cargos_permitidos = [
            "[⚜️] Sub Dono",
            "[⚖️] Conselho",
            "[🏆 ] Tester Superior",
            "[🛠️] Dev"
        ]

        if not any(
            discord.utils.get(interaction.user.roles, name=cargo)
            for cargo in cargos_permitidos
        ):
            await interaction.response.send_message(
                "❌ Você não pode fechar este ticket.",
                ephemeral=True
            )
            return

        await interaction.response.send_message(
            "🔒 Fechando ticket em 5 segundos..."
        )

        await asyncio.sleep(5)

        await interaction.channel.delete()



# ==========================
# TRATAMENTO GERAL DE ERROS
# ==========================

@bot.tree.error
async def on_app_command_error(
    interaction: discord.Interaction,
    error
):

    if isinstance(error, app_commands.MissingPermissions):

        await interaction.response.send_message(
            "❌ Você não possui permissão para utilizar este comando.",
            ephemeral=True
        )
        return

    if isinstance(error, app_commands.CommandOnCooldown):

        await interaction.response.send_message(
            f"⏳ Aguarde {error.retry_after:.1f} segundos para usar este comando novamente.",
            ephemeral=True
        )
        return

    print("ERRO:", error)

    try:
        await interaction.response.send_message(
            "❌ Ocorreu um erro inesperado.",
            ephemeral=True
        )
    except discord.InteractionResponded:
        await interaction.followup.send(
            "❌ Ocorreu um erro inesperado.",
            ephemeral=True
        )


# ==========================
# INICIAR BOT
# ==========================

if __name__ == "__main__":

    print("=" * 40)
    print(" Iniciando Bot...")
    print("=" * 40)

    bot.run(TOKEN)
