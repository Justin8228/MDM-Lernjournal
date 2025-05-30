# Projekt 2 Java

## Übersicht

| | Bitte ausfüllen |
| -------- | ------- |
| Variante | Vorlesungsbeispiel |
| Datensatz (wenn selbstgewählt) | Vorlesungsbeispiel |
| Datensatz (wenn selbstgewählt) | Vorlesungsbeispiel |
| Modell (wenn selbstgewählt) | Vorlesungsbeispiel |
| ML-Algorithmus | Lineare Regression & Gradient Boosting Regressor |
| Repo URL | https://github.com/Justin8228/djl-serving-consumer https://github.com/Justin8228/djl-sentiment-analysis https://github.com/Justin8228/djl-footwear-classification |



## Dokumentation

### Daten

Die verwendeten Schuhbilder stammen vom UT-Zap50K-Datensatz (https://vision.cs.utexas.edu/projects/finegrained/utzap50k/), der original rund 50.000 farbige, quadratische Bilder unterschiedlicher Schuhklassen enthält. Für unser Projekt wurde eine stark verkleinerte „Small“-Variante (im Verzeichnis ut-zap50k-images-square-small) mit 7.781 Bildern im JPEG-Format und einer Auflösung von 136 × 136 Pixel eingesetzt.

Die Ordnerstruktur bildet sechs Klassen ab:

- Boots: Ankle
- Sandals: Athletic
- Shoes: Boat Shoes
- Slippers: Boot, Slipper Flats, Slipper Heels

Dadurch ergibt sich eine eindeutige Label-Zuordnung und eine gute Trainingsbasis für die Klassifikation.

![Sandale](images/Sandale.jpg)

### Training

Im Modul djl-footwear-classification steuert die Klasse Training.java den vollständigen Trainingsablauf: Über die Deep Java Library (DJL) wird zunächst ein ResNet-50-Netzwerk aufgebaut und mit den vorbereiteten Schuhbildern aus ut-zap50k-images-square-small trainiert. Dabei kommen Standard-Hyperparameter wie eine Batch-Größe von 32, zwei Epochen und die Softmax-Cross-Entropy-Verlustfunktion zum Einsatz, ergänzt durch einen Accuracy-Evaluator und das eingebaute Logging von DJL.

Der Datensatz wird mit ImageFolder geladen und automatisch in ein 80/20-Split für Training und Validierung aufgeteilt. Ein typischer Ausschnitt aus der Datenvorbereitung sieht so aus:

ImageFolder dataset = ImageFolder.builder()
    .setRepositoryPath(Paths.get("ut-zap50k-images-square-small"))
    .optMaxDepth(10)
    .addTransform(new Resize(100, 100))
    .addTransform(new ToTensor())
    .setSampling(32, true)
    .build();
dataset.prepare();
RandomAccessDataset[] split = dataset.randomSplit(8, 2);


Für den eigentlichen Trainingslauf initialisiert Training.java das Modell, legt die Eingabe-Shape und Metriken fest und startet dann das Fitting:

Model model = Models.getModel();  // ResNet-50-Gerüst
Trainer trainer = model.newTrainer(
    TrainingConfig.builder()
      .optLoss(Loss.softmaxCrossEntropyLoss())
      .addEvaluator(new Accuracy())
      .addTrainingListeners(TrainingListener.Defaults.logging())
      .build()
);
trainer.initialize(new Shape(1, 3, 100, 100));
EasyTrain.fit(trainer, 2, split[0], split[1]);


Nach Abschluss sichert das Skript das finale Modell und die Synset-Datei unter models:

model.save(Paths.get("models"), "shoeclassifier");
Models.saveSynset(Paths.get("models"), dataset.getSynset());

### Inference / Serving

Im Modul djl-serving-consumer läuft der Inference-Service als Spring-Boot-Anwendung, die das zuvor trainierte ResNet-50-Modell lädt und über einen REST-Endpoint Anfragen entgegennimmt. Beim Start initialisiert die Anwendung einen DJL-Predictor, der auf alle eingehenden Bilder angewendet wird.

Ein zentraler Ausschnitt aus PredictionController.java zeigt, wie das Model beim Bootstrapping geladen und für jeden Request genutzt wird:

@RestController
public class PredictionController {

    private Predictor<Image, Classifications> predictor;

    @PostConstruct
    public void init() throws IOException, ModelNotFoundException, MalformedModelException {
        Criteria<Image, Classifications> criteria = Criteria.builder()
            .setTypes(Image.class, Classifications.class)
            .optModelPath(Paths.get("models"))
            .optTranslator(new ImageClassificationTranslator())
            .build();
        ZooModel<Image, Classifications> model = ModelZoo.loadModel(criteria);
        predictor = model.newPredictor();
    }

    @PostMapping("/predict")
    public Classifications predict(@RequestParam("file") MultipartFile file) throws IOException, TranslateException {
        Image img = ImageFactory.getInstance().fromInputStream(file.getInputStream());
        return predictor.predict(img);
    }
}


Clients senden per POST /predict ein Bild (Multipart-Upload) und erhalten eine JSON-Liste der Top-Klassifizierungen samt Wahrscheinlichkeiten zurück, zum Beispiel:

{
  "classifications": [
    { "className": "Boots", "probability": 0.87 },
    { "className": "Ankle", "probability": 0.05 },
    …
  ]
}
![Image Classification](images/Image%20Classification.png)
![Sentiment Analysis](images/Sentiment%20Analysis.png)

### Deployment

Für das Deployment wurden alle Komponenten in Docker-Container verpackt, um eine einheitliche Laufzeitumgebung sicherzustellen. In jedem Modul liegt eine eigene Dockerfile, die auf openjdk:21-jdk-slim basiert und das fertige JAR-Paket aus dem Maven-Build kopiert. Am Beispiel des Moduls djl-footwear-classification sieht das so aus:

FROM openjdk:21-jdk-slim
WORKDIR /usr/src/app
COPY models models
COPY src src
COPY .mvn .mvn
COPY pom.xml mvnw ./
RUN ./mvnw -Dmaven.test.skip=true package
EXPOSE 8080
CMD ["java","-jar","target/playground-0.0.1-SNAPSHOT.jar"]


Der Build-Befehl erfolgt im Projekt-Root jeweils mit Angabe des Modulpfads, zum Beispiel:

docker build -t mosazhaw/djl-footwear-classification djl-footwear-classification/


Analog werden daraus Images für djl-sentiment-analysis und den Consumer-Service erzeugt. Beim djl-serving-consumer-Modul existiert darüber hinaus ein eigener Unterordner djl-serving, der auf dem offiziellen DJL-Serving-Image (deepjavalibrary/djl-serving:0.31.0) aufsetzt. Dort wird das komprimierte Modell in den Standard-Pfad /opt/ml/model entpackt:

FROM deepjavalibrary/djl-serving:0.31.0
COPY *.zip /opt/ml/
RUN unzip /opt/ml/resnet18_v1.zip -d /opt/ml/model/resnet18_v1
RUN rm /opt/ml/resnet18_v1.zip


Zum Starten der beiden Services – Model-Service und Web-Service – liegt im djl-serving-consumer/djl-serving-Verzeichnis eine docker-compose.yml bereit, die beide Container als Stack hochzieht:

services:
  web-service:
    depends_on:
      - model-service
    image: mosazhaw/djl-serving-consumer:latest
    ports:
      - "80:8082"
    restart: always

  model-service:
    image: mosazhaw/djl-serving:latest
    restart: always


Die Images wurden auf Docker Hub gepusht (justin8228/djl-footwear-classification:latest), sodass sie sich problemlos in Cloud-Umgebungen oder Kubernetes-Clustern referenzieren lassen.

Nach erfolgreichem Build und Push auf Docker Hub starten wir docker-compose up -d im Ordner djl-serving-consumer/djl-serving. Im Docker Desktop sehen wir anschließend die beiden laufenden Services:
![Docker Desktop](images/Docker%20Desktop.png)

Für die Cloud-Bereitstellung verwenden wir Azure App Services. Nach CLI-Befehlen zum Erstellen der Ressourcengruppe und des Plans findet sich die Anwendung im Azure-Portal unter … Die folgenden Screenshots zeigen jeweils die Übersicht der Web-Apps mit ihrer URL, dem App Service-Plan und dem Betriebssystem.
![Azure Serving](images/azure%20serving.png)
![Azure Sentiment](images/azure%20sentiment.png)
