# Lenslate

## Descrizione  
Il progetto è stato realizzato cercando di imitare almeno parzialmente le funzionalità della celebre app Google Lens. Per farlo, sono state utilizzate tecnologie come CameraX per la gestione della fotocamera e ML Kit per l’analisi delle immagini direttamente sul dispositivo, senza necessità di connessione a servizi cloud.

A differenza di Google Lens, che sfrutta anche il machine learning basato su cloud, questa implementazione si concentra su un'elaborazione più locale, garantendo maggiore privacy e velocità. L'app è stata pensata principalmente per il riconoscimento e la traduzione di testi in tempo reale, risultando utile per studenti, viaggiatori e professionisti.

In futuro, potrebbe essere integrata una funzione di ricerca avanzata per oggetti o la possibilità di salvare e organizzare i risultati delle scansioni in una libreria interna.

## Linguaggi e Tecnologie Utilizzate  
- **Kotlin**  
- **Jetpack Compose**
- **CameraX**
- **Google Cloud**
- **Firebase**

## Funzionalità Principali  
✔️ **Scansione Codici QR** → Scansione in tempo reale di codici QR i quali se sono url vengono scansionati di nuovo per verificarne la sicurezza.  
✔️ **Riconoscimento Testo** → Permette di riconoscere del testo usando 'Vision API'.  
✔️ **Traduzione Testo** → Determinare la lingua del testo scansionato ed eventualmente tradurlo con Firebase mlKit.  
✔️ **Scansione Documenti** → Permette di usere una libreria di google per scansionare dei documenti e salvarli nell'app come file pdf.  
✔️ **Riconoscimento Landmark** → Feature per riconoscere i punti di riferimento dalla galleria o dalle foto scattate nell'app.  

## Panoramica dell'App  

<p align="start">
Schermata Principale<br/>
<img src="https://i.imgur.com/bXSgKiN.jpeg" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

## Camera Preview Composable
Implementazione della preview usando jetpack compose e cameraX:

```kotlin
@Composable
fun CameraPreview(
    modifier: Modifier = Modifier,
    viewModel: MainViewmodel,
    controller: LifecycleCameraController
) {
    val torchState by viewModel.torchState.collectAsState()
    var tapPosition by remember { mutableStateOf<Offset?>(null) }

    Box(modifier = modifier.fillMaxSize()) {
        AndroidView(factory = {
            PreviewView(it).apply {
                this.controller = controller

                this.setOnTouchListener { view, event ->
                    if (event.action == MotionEvent.ACTION_DOWN) {
                        val factory = this.meteringPointFactory
                        val meteringPoint = factory.createPoint(event.x, event.y)
                        val action = FocusMeteringAction.Builder(meteringPoint).build()
                        this.controller?.cameraControl?.startFocusAndMetering(action)

                        val viewLocation = IntArray(2)
                        view.getLocationOnScreen(viewLocation)
                        val adjustedX = event.rawX - viewLocation[0]
                        val adjustedY = event.rawY - viewLocation[1]
                        tapPosition = Offset(adjustedX, adjustedY)
                    }
                    true
                }
            }
        }, modifier = Modifier.fillMaxSize())

        tapPosition?.let { position ->
            FocusIndicator(position)
        }

        if (torchState) {
            controller.enableTorch(true)
        } else {
            controller.enableTorch(false)
        }
    }
}
```

## Scansione QR
Implementazione della scansionde del qr sull'Image Analysis Analyzer

```kotlin
controller = LifecycleCameraController(this@SecondActivity).apply {
            setEnabledUseCases(
                CameraController.IMAGE_CAPTURE or CameraController.IMAGE_ANALYSIS
            )
            setImageAnalysisAnalyzer(
                ContextCompat.getMainExecutor(applicationContext),
                QrCodeAnalyzer{
                    viewModel.OnQRanalized0(it)
                }
            )
        }
```

## QR code Analyzer

```kotlin
class QrCodeAnalyzer(
    private val onQrCodeScanned: (String) -> Unit
) : ImageAnalysis.Analyzer {

    private val supportedImageFormats = listOf(
        ImageFormat.YUV_420_888,
        ImageFormat.YUV_422_888,
        ImageFormat.YUV_444_888,
    )

    override fun analyze(image: ImageProxy) {
        if (image.format in supportedImageFormats) {
            val bytes = image.planes.first().buffer.toByteArray()

            val source = PlanarYUVLuminanceSource(
                bytes,
                image.width,
                image.height,
                0,
                0,
                image.width,
                image.height,
                false
            )

            val binaryBitmap = BinaryBitmap(HybridBinarizer(source))
            
            CoroutineScope(Dispatchers.Default).launch {
                try {
                    val result = MultiFormatReader().apply {
                        setHints(
                            mapOf(
                                DecodeHintType.POSSIBLE_FORMATS to arrayListOf(
                                    BarcodeFormat.QR_CODE
                                )
                            )
                        )
                    }.decode(binaryBitmap)

                    onQrCodeScanned(result.text)
                    Log.e("QR","THE QR HAS BEEN SCANNED")

                } catch (e: Exception) {
                    e.printStackTrace()
                } finally {
                    image.close()
                }
            }
        } else {
            image.close()
        }
    }

    private fun ByteBuffer.toByteArray() : ByteArray {
        rewind()
        return ByteArray(remaining()).also { get(it) }
    }


}
```

  
Nav Drawer<br/>
<img src="https://i.imgur.com/zQ69NXy.jpeg" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

```kotlin
ModalNavigationDrawer(
                    drawerContent = {
                        ModalDrawerSheet(drawerContainerColor = offWhite) {
                            ProfileScreen(
                                userData = googleAuthUiClient.getSignedInUser(),
                                onSignOut = {
                                    lifecycleScope.launch {
                                        googleAuthUiClient.signOut()
                                        user = googleAuthUiClient.getSignedInUser()
                                        sharedPreferences.edit().putBoolean("isLoggedIn", false).apply()
                                        Toast.makeText(context,R.string.signedOut,Toast.LENGTH_SHORT).show()
                                    }
                                    scope.launch {
                                        drawerState.close()
                                    }
                                },
                                onSignIn = {
                                    lifecycleScope.launch {
                                        try{
                                            val signInIntentSender = googleAuthUiClient.signIn()
                                            launcher.launch(
                                                IntentSenderRequest
                                                    .Builder(
                                                        signInIntentSender ?: return@launch
                                                    )
                                                    .build()
                                            )
                                        }
                                        catch (e:Exception){
                                            e.printStackTrace()
                                        }
                                    }
                                },
                                onDeleteAccount = {
                                         lifecycleScope.launch{
                                             val isDeleted = googleAuthUiClient.deleteUserDataAndAccount()
                                             if (isDeleted) {
                                                 googleAuthUiClient.signOut()
                                                 user = googleAuthUiClient.getSignedInUser()
                                                 sharedPreferences.edit().putBoolean("isLoggedIn", false).apply()
                                                 Toast.makeText(context, R.string.deleteString, Toast.LENGTH_SHORT).show()
                                             } else {
                                                 Toast.makeText(context, R.string.failedString, Toast.LENGTH_SHORT).show()
                                             }
                                         }
                                },
                                activity = this@SecondActivity,
                                context = applicationContext
                            )
                        }
                    },
                    drawerState = drawerState
                )
```


Custom Image Editor<br/>
<img src="https://i.imgur.com/AVB2AVr.jpeg" height="80%" width="40%" alt="Schermata App"/>
<br />

```kotlin

 var rectWidth by remember { mutableFloatStateOf(0f) }
    var rectHeight by remember { mutableFloatStateOf(0f) }

    var offsetX by remember { mutableFloatStateOf(0f) }
    var offsetY by remember { mutableFloatStateOf(0f) }

    var sizeWidth by remember { mutableFloatStateOf(0f) }
    var sizeHeight by remember { mutableFloatStateOf(0f) }

    var lastDragAmount by remember { mutableStateOf(Offset.Zero) }
    var isDragging by remember { mutableStateOf(false) }


 Column(
                                modifier = Modifier
                                    .fillMaxWidth()
                                    .padding(5.dp),
                                horizontalAlignment = Alignment.CenterHorizontally
                            ) {

                                Box(
                                    modifier = Modifier
                                        .clip(RoundedCornerShape(15.dp))
                                ) {

                                    if (bitmap.isNotEmpty()) {
                                        val currentBitmap = scaleBitmapIfNeeded(bitmap.last(),filePath).asImageBitmap()
                                        Image(
                                            bitmap = currentBitmap,
                                            contentDescription = null
                                        )

                                        Canvas(
                                            modifier = Modifier.matchParentSize()
                                        ) {
                                            sizeWidth = size.width
                                            sizeHeight = size.height

                                            val strokeWidth = 8f
                                            rectWidth = cropWidth * size.width / currentBitmap.width
                                            rectHeight = cropHeight * size.height / currentBitmap.height

                                            if (!offsetFirstLook) {
                                                offsetX =
                                                    ((size.width - rectWidth) / 2).coerceIn(
                                                        0f,
                                                        size.width - rectWidth
                                                    )
                                                offsetY = ((size.height - rectHeight) / 2).coerceIn(
                                                    0f,
                                                    size.height - rectHeight
                                                )
                                                offsetFirstLook = true
                                            }

                                            offsetX = offsetX.coerceIn(0f, size.width - rectWidth)
                                            offsetY = offsetY.coerceIn(0f, size.height - rectHeight)


                                            drawRoundRect(
                                                color = ROIcolor,
                                                topLeft = Offset(offsetX, offsetY),
                                                size = Size(rectWidth, rectHeight),
                                                style = Stroke(width = strokeWidth),
                                                cornerRadius = CornerRadius(15f, 15f)
                                            )


                                            drawRoundRect(
                                                color = Color.Transparent,
                                                topLeft = Offset(
                                                    offsetX + strokeWidth,
                                                    offsetY + strokeWidth
                                                ),
                                                size = Size(
                                                    rectWidth - 2 * strokeWidth,
                                                    rectHeight - 2 * strokeWidth
                                                ),
                                                cornerRadius = CornerRadius(15f, 15f)
                                            )
                                        }

                                        Icon(
                                            imageVector = Icons.Default.ControlCamera,
                                            contentDescription = "Move offset",
                                            tint = ROIcolor,
                                            modifier = Modifier
                                                .size(30.dp)
                                                .offset(
                                                    x = (offsetX / d.density).dp,
                                                    y = (offsetY / d.density).dp
                                                )
                                                .pointerInput(Unit) {
                                                    detectDragGestures(
                                                        onDragStart = { isDragging = true },
                                                        onDragEnd = {
                                                            isDragging = false
                                                            lastDragAmount = Offset.Zero
                                                        }
                                                    ) { _, dragAmount ->
                                                        if (isDragging) {
                                                            val original = Offset(offsetX, offsetY)
                                                            val summed = original + dragAmount

                                                            offsetX = summed.x
                                                            offsetY = summed.y
                                                        }
                                                    }
                                                }
                                        )
                                    }

                                }

                                Row(
                                    verticalAlignment = Alignment.CenterVertically,
                                    modifier = Modifier
                                        .fillMaxWidth()
                                        .padding(10.dp)
                                ) {
                                    Text(
                                        text = stringResource(id = R.string.width),
                                        fontWeight = FontWeight.Bold,
                                        style = TextStyle(color = Color.White)
                                    )

                                    Spacer(modifier = Modifier.width(5.dp))

                                    Slider(
                                        value = cropWidth,
                                        onValueChange = {
                                            cropWidth = it
                                        },
                                        colors = SliderDefaults.colors(
                                            thumbColor = colorTertiary,
                                            activeTrackColor = colorTertiary,
                                            inactiveTrackColor = colorFourth
                                        ),
                                        valueRange = 0f..image.width.toFloat(),
                                        modifier = Modifier.weight(1f)
                                    )
                                }

                                Row(
                                    verticalAlignment = Alignment.CenterVertically,
                                    modifier = Modifier
                                        .fillMaxWidth()
                                        .padding(10.dp)
                                ) {
                                    Text(
                                        text = stringResource(id = R.string.height),
                                        fontWeight = FontWeight.Bold,
                                        style = TextStyle(color = Color.White)
                                    )

                                    Spacer(modifier = Modifier.width(5.dp))

                                    Slider(
                                        value = cropHeight,
                                        onValueChange = {
                                            cropHeight = it
                                        },
                                        colors = SliderDefaults.colors(
                                            thumbColor = colorTertiary,
                                            activeTrackColor = colorTertiary,
                                            inactiveTrackColor = colorFourth
                                        ),
                                        valueRange = 0f..image.height.toFloat(),
                                        modifier = Modifier.weight(1f)
                                    )
                                }

                            }

```

Scansione Testo e Traduzione<br/>
<img src="https://i.imgur.com/2UWTaMr.jpeg" height="80%" width="40%" alt="Schermata App"/>
<br />

## Analisi Immagine (Testo e Landscape)

```kotlin
private fun imageProcess(
    context: Context,
    bitmap: Bitmap,
    viewModel: MainViewmodel,
    toolInt : Int,
    userState: Boolean,
    userId: String?,
    scope : CoroutineScope
){

    val image = FirebaseVisionImage.fromBitmap(bitmap)

    if (toolInt == 1){
        val languageIdentifier = LanguageIdentification.getClient(
            LanguageIdentificationOptions.Builder()
                .setConfidenceThreshold(0.90f)
                .build()
        )

        val cloudTextRecognizer = FirebaseVision.getInstance()
            .cloudDocumentTextRecognizer

        val qrAnalyzer = FirebaseVision.getInstance()
            .visionBarcodeDetector

        qrAnalyzer.detectInImage(image)
            .addOnSuccessListener { qrResult ->
                if(qrResult.isNotEmpty()){
                    val barcodeStrings = qrResult.mapNotNull { it.rawValue }
                    val qrString = barcodeStrings.first()
                    viewModel.OnQRanalized(qrString)
                }
                else{
                    val emptyString = ""
                    viewModel.OnQRanalized(emptyString)
                }
            }
            .addOnFailureListener{
                val emptyString = ""
                viewModel.OnQRanalized(emptyString)
            }


        cloudTextRecognizer.processImage(image)
            .addOnSuccessListener { text ->

                val textRecognized = text.text

                languageIdentifier.identifyLanguage(textRecognized)
                    .addOnSuccessListener { language ->
                        viewModel.onLanguageRecognized(language)
                    }
                viewModel.OnTakePhotoLine(textRecognized)

                if (textRecognized.isNotBlank()){
                    if (userState){
                        if (userId != null){
                            scope.launch {
                                writeStringData(userId, history = "history1",textRecognized,context)
                            }
                        }
                    }
                }

            }
            .addOnFailureListener { e ->
                val emptyString = ""
                Toast.makeText(context,R.string.somethingWrong, Toast.LENGTH_LONG).show()
                viewModel.OnTakePhotoLine(emptyString)
            }
    }

    if(toolInt == 3){

        val detector = FirebaseVision.getInstance()
            .visionCloudLandmarkDetector

        detector.detectInImage(image)
            .addOnSuccessListener { firebaseVisionCloudLandmarks ->
                if (firebaseVisionCloudLandmarks.isNotEmpty()) {
                    for (landmark in firebaseVisionCloudLandmarks) {
                        val landmarkName = landmark.landmark
                        val confidence = landmark.confidence
                        val locations = landmark.locations
                        for (location in locations) {
                            val latitude = location.latitude
                            val longitude = location.longitude
                            viewModel.addLandmarkName(landmarkName, lat = latitude.toString(), long = longitude.toString())
                        }
                    }
                } else {
                    val emptyString = ""
                    viewModel.addLandmarkName(emptyString,emptyString,emptyString)
                }
            }
            .addOnFailureListener { e ->
                Toast.makeText(context, R.string.somethingWrong, Toast.LENGTH_LONG).show()
            }
    }

}
```

## Traduzione testo

```kotlin
suspend fun translate(
    finalText: String,
    scannedLanguage: String,
    targetLanguage: String,
    onLanguageTranslated : (String) -> Unit,
    context: Context,
    oneTime: Int,
    userState: Boolean,
    userId: String?,
    scope: CoroutineScope
): Result<String> {


    return try {
        withContext(Dispatchers.IO) {
            val sourceLangCode = TranslateLanguage.fromLanguageTag(scannedLanguage)
            val targetLangCode = TranslateLanguage.fromLanguageTag(targetLanguage)

            val ai: ApplicationInfo = context.packageManager
                .getApplicationInfo(context.packageName, PackageManager.GET_META_DATA)
            val value = ai.metaData["com.google.translation.API_KEY"]
            val apiKey = value.toString()


            val translate = TranslateOptions.newBuilder()
                .setApiKey(apiKey)
                .setTransportOptions(TranslateOptions.getDefaultHttpTransportOptions())
                .build()
                .service

            val translation = translate.translate(
                finalText,
                Translate.TranslateOption.targetLanguage(targetLangCode),
                Translate.TranslateOption.sourceLanguage(sourceLangCode)
            )
            val translatedText: String = translation.translatedText
            val decodedTranslation = decodeHtmlEntities(translatedText)
            onLanguageTranslated(decodedTranslation)
            if (userState){
                if (userId != null){
                    scope.launch {
                        writeStringData(userId, history = "history2",decodedTranslation,context)
                    }
                }
            }
            Result.success(decodedTranslation)
        }
    } catch (e: TranslateException) {

        Log.e("TextRecognition", "Text recognition failed: ${e.message}", e)

        withContext(Dispatchers.Main) {
            if (oneTime == 0){
                Toast.makeText(context, R.string.networkError, Toast.LENGTH_SHORT).show()
            }
        }
        Result.failure(e)
    } catch (e: NetworkOnMainThreadException) {

        Log.e("TextRecognition", "Text recognition failed: ${e.message}", e)

        withContext(Dispatchers.Main) {
            if (oneTime == 0){
                Toast.makeText(context, R.string.networkError, Toast.LENGTH_SHORT).show()
            }
        }
        Result.failure(e)
    }
}
```

</p>

<!--
 ```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
--!>
