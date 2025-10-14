from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    Filters,
    CallbackContext,
)
from PIL import Image
import io
import PyPDF2  # For handling PDF files

BOT_TOKEN = "7972401234:AAGjGkm49oh_4JCy6S1Q3abmKwJ4NdhGZpc"  # Replace with your actual bot token

async def start(update: Update, context: CallbackContext.DEFAULT_TYPE) -> None:
    """Starts the bot and explains its purpose."""
    await update.message.reply_text(
        "Welcome! Send me a PDF containing an ID, and I'll return a high-quality image of it."
    )

async def handle_pdf(update: Update, context: CallbackContext.DEFAULT_TYPE) -> None:
    """Handles the PDF file and extracts the image."""
    try:
        pdf_file = await update.message.document.get_file()
        pdf_bytes = await pdf_file.download_as_bytearray()

        # Extract image from the PDF
        image_stream = extract_image_from_pdf(io.BytesIO(bytes(pdf_bytes)))

        if image_stream:
            await update.message.reply_photo(photo=image_stream, caption="Here's your ID image!")
        else:
            await update.message.reply_text("Could not extract an image from the PDF.")

    except Exception as e:
        print(f"Error processing PDF: {e}")
        await update.message.reply_text(
            "An error occurred while processing the PDF. Please make sure the file is a valid PDF and contains an image."
        )

def extract_image_from_pdf(pdf_stream: io.BytesIO) -> io.BytesIO:
    """Extracts an image from the first page of a PDF."""
    try:
        pdf_reader = PyPDF2.PdfReader(pdf_stream)
        page = pdf_reader.pages[0]  # Get the first page

        # Attempt to extract image - Improved robustness
        if '/XObject' in page['/Resources']:
            xObject = page['/Resources']['/XObject'].resolve()
            for obj in xObject:
                if xObject[obj]['/Subtype'] == '/Image':
                    try:
                        image_data = xObject[obj]._data
                        img = Image.open(io.BytesIO(image_data))

                        # Convert image to PNG and store in BytesIO
                        image_stream = io.BytesIO()
                        img.save(image_stream, format='PNG')  # Save as PNG
                        image_stream.seek(0)
                        return image_stream
                    except Exception as img_err:
                        print(f"Error processing image object {obj}: {img_err}") #More specific error

        return None # If we don't find an image.

    except Exception as e:
        print(f"Error extracting image: {e}")
        return None

def main() -> None:
    """Runs the bot."""
    application = Application.builder().token(BOT_TOKEN).build()

    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(Filters.document, handle_pdf))

    application.run_polling()

if __name__ == "__main__":
    main()
