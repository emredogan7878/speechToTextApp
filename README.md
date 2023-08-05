# Speech To Text App with Python, Amazon Transcribe and Google Speech To Text API usage report.
* These codes were written using two different speech recognition (speech-to-text) services, Google Cloud Speech-to-Text and Amazon Transcribe, to convert speech in certain audio files to text and store these texts in a database.

# google_speech_to_text.py :
```python
import wave
from google.cloud import speech

def google_speech_to_text(file_name):
    client = speech.SpeechClient.from_service_account_file('key.json')

    with open(file_name, 'rb') as f:
        wav_data = f.read()

    audio_file = speech.RecognitionAudio(content=wav_data)
    config = speech.RecognitionConfig(
        encoding="LINEAR16",
        language_code="en-US",
        sample_rate_hertz=44100,
        audio_channel_count=1,
    )

    response = client.recognize(config=config, audio=audio_file)

    transcript = ""
    confidence_sum = 0.0
    num_sentences = 0

    for result in response.results:
        transcript += result.alternatives[0].transcript + "."
        confidence_sum += result.alternatives[0].confidence
        num_sentences += 1

    average_confidence = confidence_sum / num_sentences if num_sentences > 0 else 0.0

    transcript = ". ".join(sentence.capitalize() for sentence in transcript.split(". "))

    with wave.open(file_name, 'rb') as audio_wave:
        audio_duration = audio_wave.getnframes() / audio_wave.getframerate()

    return transcript, average_confidence, int(audio_duration) 
```
* This module utilizes the Google Cloud Speech-to-Text service to convert audio files into text.
* The "google_speech_to_text" function takes an audio file as the "file_name" parameter and sends it to the Google Cloud API.
* The API response includes the transcribed text of the audio file and the confidence level of the transcription process.
* Additionally, the duration of the audio file is calculated and returned.
* As a result, the function returns a tuple containing the transcribed text, the average confidence level, and the duration of the audio file.

# amazon_speech_to_text.py :
```python
import time
import boto3
import requests

def get_transcript_text(json_url):
    response = requests.get(json_url)
    if response.status_code == 200:
        json_data = response.json()
        transcript_text = json_data['results']['transcripts'][0]['transcript']
        return transcript_text
    else:
        print("Failed to fetch transcript text.")
        return None

def get_average_confidence(json_url):
    response = requests.get(json_url)
    if response.status_code == 200:
        json_data = response.json()
        items = json_data['results']['items']
        sentence_confidences = []
        current_sentence = []
        for item in items:
            if item['type'] == 'pronunciation':
                current_sentence.append(item)
            elif item['type'] == 'punctuation' and item['alternatives'][0]['confidence'] is not None:
                sentence_confidence = sum(float(word['alternatives'][0]['confidence']) for word in current_sentence) / len(current_sentence)
                sentence_confidences.append(sentence_confidence)
                current_sentence = []
        average_confidence = sum(sentence_confidences) / len(sentence_confidences) if sentence_confidences else 0.0
        average_confidence_formatted = f"{average_confidence:.2f}".replace('.', ',')
        average_confidence_formatted = average_confidence_formatted.replace(',', '.')
        float_average_confidence = float(average_confidence_formatted)
        return float_average_confidence
    else:
        print("Failed to fetch transcript text.")
        return None

def get_total_duration(json_url):
    response = requests.get(json_url)
    if response.status_code == 200:
        json_data = response.json()
        items = json_data['results']['items']
        if items:
            for item in reversed(items):
                if 'end_time' in item:
                    total_duration = float(item['end_time'])
                    # Format total_duration with comma instead of a period
                    total_duration_formatted = f"{total_duration:.3f}".replace('.', ',')
                    total_duration_formatted = total_duration_formatted.replace(',', '.')
                    float_total_duration_formatted = float(total_duration_formatted)
                    return total_duration_formatted
            print("Failed to find 'end_time' in the transcript.")
            return None
        else:
            print("No items found in the transcript.")
            return None
    else:
        print("Failed to fetch transcript text.")
        return None
def amazon_speech_to_text(file_uri):
    transcribe_client = boto3.client('transcribe', region_name='eu-central-1')
    job_name = "Example-job6"

    transcribe_client.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={
            'MediaFileUri': file_uri
        },
        MediaFormat='wav',
        LanguageCode='en-US'
    )

    max_tries = 60
    while max_tries > 0:
        max_tries -= 1
        job = transcribe_client.get_transcription_job(TranscriptionJobName=job_name)
        job_status = job['TranscriptionJob']['TranscriptionJobStatus']
        if job_status in ['COMPLETED', 'FAILED']:
            print(f"Job Example-job578 is {job_status}.")
            if job_status == 'COMPLETED':
                transcript_file_uri = job['TranscriptionJob']['Transcript']['TranscriptFileUri']
                transcript_text = get_transcript_text(transcript_file_uri)
                if transcript_text:
                    average_confidence = get_average_confidence(transcript_file_uri)
                    total_duration = get_total_duration(transcript_file_uri)
                    if total_duration is not None:
                        # Dönüşüm öncesinde "., " karakterlerini düzeltme
                        total_duration = total_duration.replace(",", ".")
                        amazon_duration = int(float(total_duration))
            break
        else:
            print(f"Waiting for Example-job. Current status is {job_status}.")
        time.sleep(10)

    return transcript_text, average_confidence, amazon_duration
```
* This module utilizes the Amazon Transcribe service to convert audio files into text.
* The "get_transcript_text", "get_average_confidence", and "get_total_duration" functions make API requests to Amazon Transcribe to obtain the transcribed text, average confidence level, and duration of the audio file, respectively.
* The "amazon_speech_to_text" function takes an audio file URL as the "file_uri" parameter and sends it to the Amazon Transcribe API.
* The API response includes the transcribed text and the average confidence level, along with other information.
* The duration of the audio file is also calculated and returned.
* In the end, the function returns the transcribed text, average confidence level, and duration of the audio file.

# main.py :
```python
import pypyodbc
from google_speech_to_text import google_speech_to_text
from amazon_speech_to_text import amazon_speech_to_text

def update_database(voice_name, google_transcript, google_confidence, google_duration, amazon_transcript, amazon_confidence, amazon_duration):
    # Veritabanına bağlanma
    db = pypyodbc.connect(
        'DRIVER={ODBC Driver 17 for SQL Server};SERVER=127.0.0.1,1433;DATABASE=SpeechToText;UID=sa;PWD=Sistas2015development')
    imlec = db.cursor()
    select_query = '''SELECT * FROM voice WHERE voice_name = ?'''
    imlec.execute(select_query, (voice_name,))
    existing_data = imlec.fetchone()
    if existing_data:
        # Veri mevcut ise güncelleme yapma
        update_query = '''UPDATE voice
                          SET google_transcript = ?,
                              google_confidence = ?,
                              google_duration = ?,
                              amazon_transcript = ?,
                              amazon_confidence = ?,
                              amazon_duration = ?
                          WHERE voice_name = ?'''
        imlec.execute(update_query, (google_transcript, google_confidence, google_duration, amazon_transcript, amazon_confidence, amazon_duration, voice_name,))
    else:
        # Veri yoksa yeni bir satır oluşturma
        insert_query = '''INSERT INTO voice (voice_name, google_transcript, google_confidence, google_duration, amazon_transcript, amazon_confidence, amazon_duration)
                          VALUES (?, ?, ?, ?, ?, ?, ?)'''
        imlec.execute(insert_query, (voice_name, google_transcript, google_confidence, google_duration, amazon_transcript, amazon_confidence, amazon_duration))
    # Değişiklikleri kaydet ve bağlantıyı kapat
    db.commit()
    db.close()

def main():
    
    google_file_name = "deneme50sn.wav"
    google_transcript, google_confidence, google_duration = google_speech_to_text(google_file_name)

    print(f"Google Transcript: {google_transcript}")
    print(f"Google Confidence: {google_confidence}")
    print(f"Google Duration: {google_duration}")

    amazon_file_uri = 's3://transcribebucket78/deneme50sn.wav'
    amazon_transcript, amazon_confidence, amazon_duration = amazon_speech_to_text(amazon_file_uri)

    print(f"Amazon Transcript: {amazon_transcript}")
    print(f"Amazon Confidence: {amazon_confidence}")
    print(f"Amazon Duration: {amazon_duration}")

    # Veritabanına yazdırma
    voice_name = "deneme50sn.wav"
    update_database(voice_name, google_transcript, google_confidence, google_duration, amazon_transcript, amazon_confidence, amazon_duration)

if __name__ == '__main__':
    main()
```
* This file utilizes both Google and Amazon Speech-to-Text APIs to convert audio files into text and stores the results in a database.
* First, a connection is established to a SQL Server database using the pypyodbc library.
* Then, text, confidence level, and duration information are obtained from the "google_speech_to_text" and "amazon_speech_to_text" modules.
* If the added data exists in the database, its changed properties are updated. If not available, it is added.
* The results from both Google and Amazon are saved in the database.
* This process ensures that the transcribed text, confidence levels, and duration of audio files are stored in the database and can be accessed when needed.
* The main function determines the sample audio file name and calls the Google Cloud and Amazon Transcribe functions to retrieve the text, trust level, and duration information for each service. It then saves this information to the database.


* When we run the application, output be like this:
![github1111](https://github.com/emredogan7878/speechToTextApp/assets/112003747/2e05f62c-8300-46e6-9eaf-24e0b9040cbf)


* And our table called "voice" in our database will look like this (deneme50sn.wav).
![github2222](https://github.com/emredogan7878/speechToTextApp/assets/112003747/7d42ae73-7401-4b66-a339-3beb5e66e751)


