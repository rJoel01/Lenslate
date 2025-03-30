![image](https://github.com/user-attachments/assets/2922998b-76be-4a0d-8728-5e01b60e358f)# Lenslate

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
<img src="https://i.imgur.com/bXSgKiN.jpeg" height="80%" width="30%" alt="Schermata App"/>
<br /><br />

## Camera Preview Composable
The following code implements a Camera Preview using Jetpack Compose and CameraX:

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
  
Nav Drawer<br/>
<img src="https://i.imgur.com/zQ69NXy.jpeg" height="80%" width="30%" alt="Schermata App"/>
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


Image Editor<br/>
<img src="https://i.imgur.com/zQ69NXy.jpeg" height="80%" width="30%" alt="Schermata App"/>
<br />
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
