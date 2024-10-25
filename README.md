
import tweepy
from transformers import pipeline
import random
import time

class AsistenteVirtual:
    def __init__(self, nombre):
        self.nombre = nombre
        self.nlp = pipeline("text-generation", model="dccuchile/bert-base-spanish-wwm-uncased")
        self.sentiment_analyzer = pipeline("sentiment-analysis", model="nlptown/bert-base-multilingual-uncased-sentiment")
        self.temas_sensibles = ["política", "religión", "dinero"]
        self.configure_twitter()

    def configure_twitter(self):
        auth = tweepy.OAuthHandler("CONSUMER_KEY", "CONSUMER_SECRET")
        auth.set_access_token("ACCESS_TOKEN", "ACCESS_TOKEN_SECRET")
        self.api = tweepy.API(auth)

    def generar_respuesta(self, mensaje):
        if self.es_tema_sensible(mensaje):
            return self.respuesta_segura()
        sentimiento = self.analizar_sentimiento(mensaje)
        contexto = f"El usuario dice: '{mensaje}'. El sentimiento es {sentimiento}. {self.nombre} responde:"
        respuesta = self.nlp(contexto, max_length=50, num_return_sequences=1)[0]['generated_text']
        return respuesta.split(f"{self.nombre} responde:")[-1].strip()

    def es_tema_sensible(self, mensaje):
        return any(tema in mensaje.lower() for tema in self.temas_sensibles)

    def respuesta_segura(self):
        respuestas = [
            "Ese es un tema complejo. ¿Quizás podríamos hablar de algo más ligero?",
            "Prefiero no opinar sobre ese tema. ¿Qué te parece si hablamos de tus hobbies?",
            "Entiendo que es un tema importante. Sin embargo, como asistente virtual, no puedo proporcionar opiniones sobre eso."
        ]
        return random.choice(respuestas)

    def analizar_sentimiento(self, texto):
        resultado = self.sentiment_analyzer(texto)[0]
        return resultado['label']

    def publicar_tweet(self, mensaje):
        try:
            self.api.update_status(mensaje)
            print(f"Tweet publicado: {mensaje}")
        except Exception as e:
            print(f"Error al publicar tweet: {e}")

    def responder_menciones(self):
        menciones = self.api.mentions_timeline()
        for mencion in menciones:
            respuesta = self.generar_respuesta(mencion.text)
            try:
                self.api.update_status(
                    f"@{mencion.user.screen_name} {respuesta}",
                    in_reply_to_status_id=mencion.id
                )
                print(f"Respondido a @{mencion.user.screen_name}")
            except Exception as e:
                print(f"Error al responder mención: {e}")

    def ejecutar(self):
        while True:
            self.responder_menciones()
            self.publicar_tweet(f"¡Hola! Soy {self.nombre}, tu asistente virtual. ¿En qué puedo ayudarte hoy?")
            time.sleep(3600)  # Espera 1 hora antes de la próxima iteración

if __name__ == "__main__":
    asistente = AsistenteVirtual("Luna")
    asistente.ejecutar()
