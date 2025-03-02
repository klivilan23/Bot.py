# Bot.py
import logging
import time
from datetime import datetime, timedelta
import asyncio
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

# Defina seu token do BotFather (recomenda-se armazená-lo em variáveis de ambiente)
TOKEN = '6700961816:AAE92A5KjDmruwUsUd_Urmh8Y9TlH5OGMd0'

# Ativando o log para debug
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)
logger = logging.getLogger(_name_)

async def process_file(file_name, keyword):
    """ Processa o arquivo e retorna as linhas contendo a palavra-chave de forma otimizada. """
    start_time = time.time()
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    try:
        # Rodar a busca em uma thread separada para não travar o bot
        linhas = await asyncio.to_thread(search_in_file, file_name, keyword)

        if not linhas:
            return f"⚠️ Nenhum login encontrado para '{keyword}⚠️'."

        execution_time = timedelta(seconds=(time.time() - start_time))
        
        resultado = "\n".join(list(linhas)[:100])
        resposta = (
            f"📅 Data e Hora da Puxada: {now}\n"
            f"🔢 Total de Linhas Encontradas: {len(linhas)}\n"
            f"⏳ Tempo de Execução: {execution_time}\n\n"
            f"Resultados:\n{resultado}"
        )
        
        if len(linhas) > 100:
            resposta += f"\n\n⚠️ Exibindo 100 de {len(linhas)} linhas."

        return resposta

    except FileNotFoundError:
        return f"❌ Erro: Arquivo '{file_name}' não encontrado."
    except PermissionError:
        return f"❌ Erro: Sem permissão para acessar '{file_name}'."
    except UnicodeDecodeError:
        return "❌ Erro: O arquivo não está no formato UTF-8."
    except Exception as e:
        return f"❌ Erro inesperado: {e}"

def search_in_file(file_name, keyword):
    """ Função síncrona para buscar no arquivo (será rodada em outra thread). """
    with open(file_name, 'r', encoding='utf-8') as infile:
        return {linha.strip() for linha in infile if keyword.lower() in linha.lower()}

# Função de comando /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("💻Digite sua URL 💻")

# Função para lidar com a palavra-chave
async def handle_keyword(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    keyword = update.message.text.strip()
    file_name = "/storage/emulated/0/Documents/2.txt"  # Caminho do arquivo

    loading_message = await update.message.reply_text("🔍 Buscando...")

    # Processa a busca de forma assíncrona
    response_text = await process_file(file_name, keyword)

    # Edita a mensagem para exibir os resultados
    await loading_message.edit_text(response_text)

# Função principal para rodar o bot
def main():
    application = Application.builder().token(TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_keyword))

    application.run_polling()

if _name_ == '_main_':
    main()
