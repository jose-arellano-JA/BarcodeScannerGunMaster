## Android  Escáner de códigos de barras (lectura mediante teclado externo)

### 1. Resumen general
Se conecta un escáner de códigos de barras estándar a un dispositivo Android para extraer el contenido escaneado por el escáner. Se coloca un EditTextView en la interfaz para mostrar directamente el contenido en el EditTextView. Sin embargo, en algunas interfaces no se puede colocar un EditTextView. A continuación se describe el procedimiento para las interfaces que no tienen EditTextView. El escáner de códigos de barras y el teclado externo funcionan según el mismo principio, por lo que se ha investigado también el teclado externo.

Traducción realizada con la versión gratuita del traductor DeepL.com
### 2. Escáner de códigos de barras - Dispositivo de entrada
n el proyecto se utiliza un escáner de códigos de barras estándar (el modelo NLS-FR40 de Newland). Por «estándar» se entiende que no se proporcionan documentos de desarrollo. Tras investigar, se descubrió que utiliza el estándar «evento de entrada», igual que un teclado externo. Al tratarse de un evento de entrada, se utiliza el método dispatchKeyEvent de Activity.
```@Override
    public boolean dispatchKeyEvent(KeyEvent event) {
```

Cuando el escáner de códigos de barras reconoce el código escaneado, se generan dos eventos: KEYCODE_ENTER y KEYCODE_DPAD_DOWN. En la documentación consultada se menciona KEYCODE_ENTER, pero no se hace referencia a KEYCODE_DPAD_DOWN, y tampoco se sabe si otros escáneres de códigos de barras generan este evento.
### 3. Resultados del experimento
En el método dispatchKeyEvent de Activity, se imprimió el registro de KeyEvent: (solo se imprimió el registro de acción = ACTION_UP, se ignoró la acción = ACTION_DOWN).

- 3.1. Registro del teclado virtual de los dispositivos Android.
![在这里插入图片描述](image/inputkey.png)

- 3.2. Registro del escáner externo (NLS-FR40 de Newland)
 ![在这里插入图片描述](image/scanner.png)

- 3.2. Registro del teclado externo (teclado estándar)
![在这里插入图片描述](image/keyboard.png)
   Aquí va una nota: si la tecla Num del teclado numérico está bloqueada, metaState = meta_num_lock_on.

Resumen de conclusiones comparativas:
1. El escáner externo estándar y el teclado externo estándar son dispositivos de entrada similares.
2. En los eventos de entrada del disco de software incluido, los valores de deviceId, source, scanCode y flag difieren de los de los dispositivos externos.
3. Los valores de deviceId y source del escáner y del teclado externo son diferentes.


### 4. Revisar el código fuente de KeyEvent para comparar.
Al examinar el código fuente de keyEvent, se puede observar claramente que el teclado virtual del dispositivo tiene el valor deiveId=KeyCharacterMap.VIRTUAL_KEYBOARD (-1) fijado de forma estática. Por lo tanto, se utiliza la diferencia en este campo para distinguir entre el teclado virtual y el teclado externo. Actualmente no se tiene previsto distinguir entre el escáner y el teclado externo.
```/**
     * Create a new key event.
     *
     * @param downTime El tiempo (en {@link android.os.SystemClock#uptimeMillis})
     * en el que este código de tecla se desactivó originalmente.
     * @param eventTime El tiempo (en {@link android.os.SystemClock#uptimeMillis})
     * en el que ocurrió este evento.
     * @param action Código de acción: {@link #ACTION_DOWN},
     * {@link #ACTION_UP} o {@link #ACTION_MULTIPLE}.
     * @param code El código de tecla.
     * @param repeat Un recuento de repeticiones para eventos de pulsación (> 0 si es después de la
     * pulsación inicial) o un recuento de eventos para eventos múltiples.
     */
    public KeyEvent(long downTime, long eventTime, int action,
                    int code, int repeat) {
        mDownTime = downTime;
        mEventTime = eventTime;
        mAction = action;
        mKeyCode = code;
        mRepeatCount = repeat;
        mDeviceId = KeyCharacterMap.VIRTUAL_KEYBOARD;
    }

    

    /**
     * Crear un nuevo evento de tecla.
     *
     * @param downTime El tiempo (en {@link android.os.SystemClock#uptimeMillis})
     * en el que este código de tecla se pulsó originalmente.
     * @param eventTime El tiempo (en {@link android.os.SystemClock#uptimeMillis})
     * en el que se produjo este evento.
     * @param action Código de acción: {@link #ACTION_DOWN},
     * {@link #ACTION_UP} o {@link #ACTION_MULTIPLE}.
     * @param code El código de tecla.
     * @param repeat Un recuento de repeticiones para eventos de pulsación (> 0 si es después de la
     * pulsación inicial) o un recuento de eventos para eventos múltiples.
     * @param metaState Indicadores que muestran qué teclas meta están pulsadas actualmente.
     * @param deviceId El ID del dispositivo que generó el evento de tecla.
     * @param scancode Código de exploración del dispositivo sin procesar del evento.
     */
    public KeyEvent(long downTime, long eventTime, int action,
                    int code, int repeat, int metaState,
                    int deviceId, int scancode) {
        mDownTime = downTime;
        mEventTime = eventTime;
        mAction = action;
        mKeyCode = code;
        mRepeatCount = repeat;
        mMetaState = metaState;
        mDeviceId = deviceId;
        mScanCode = scancode;
    }
```
### 5, Estrategia de interceptación
Se requieren algunos conocimientos básicos sobre la «transmisión de eventos» en Android, algo imprescindible para una entrevista de trabajo. Ya lo he mencionado anteriormente:[Android 事件传递与焦点处理(tv)](https://blog.csdn.net/lckj686/article/details/44858387)
La transmisión de eventos en Activity, especialmente la interceptación de teclas, es muy conveniente. Basta con reescribir el método dispatchKeyEvent. La idea de la reescritura también es muy sencilla: se determina si se trata de un escáner utilizando deviceId == -1.
Pseudocódigo

```@Override
   public boolean dispatchKeyEvent(KeyEvent event) {
        Log.d(TAG, "event= " + event);

        if (Si se trata de un incidente relacionado con un escáner de código de barras.) {
         //Se consume directamente, sin transmitirse hacia abajo. El editTextView tampoco se rellena automáticamente, y el evento KEYCODE_ENTER tampoco afecta a otros controles, como el evento de clic del botón.
            return true;
        }

        return super.dispatchKeyEvent(event);
    }
```
En la práctica, las cosas no suelen ser tan drásticas. Por ejemplo, se puede controlar si se bloquea por completo o no, o encapsular por separado las herramientas de gestión. Estas son técnicas de encapsulación, y al final del capítulo se ofrece un ejemplo sencillo de encapsulación.
```/**
     * 处理输入事件
     *
     * @param event
     * @return true Indica que se ha consumido, la interceptación no se transmite, false no importa.
     */
    public boolean dispatchKeyEvent(KeyEvent event) {

        /**
         * El teclado virtual del sistema: al pulsarlo, se muestra -1, sin importar, no se intercepta.
         */
        if (event.getDeviceId() == -1) {
            return false;
        }

        //Al pulsar y soltar, si se detecta el movimiento de pulsar y soltar, se cuenta como una entrada válida.
        //Cualquier evento relacionado con el escáner de códigos de barras debe ser procesado; de lo contrario, se mostrará en el campo de texto editable.
        if (event.getAction() == KeyEvent.ACTION_UP) {

            //Solo números, no hay letras en el código unidimensional.
            int code = event.getKeyCode();
            if (code >= KeyEvent.KEYCODE_0 && code <= KeyEvent.KEYCODE_9) {

                codeStr += (code - KeyEvent.KEYCODE_0);
            }

            //Una vez finalizada la identificación, el dispositivo actualmente en uso es  . También se producirá un evento KEYCODE_DPAD_DOWN. No sé si otros dispositivos también lo tienen.  Ignóralo por ahora.
            if (code == KeyEvent.KEYCODE_ENTER) {

                if (listener != null) {
                    listener.onResult(codeStr);
                    codeStr = "";
                }
            }

        }
        //Todos los incidentes relacionados con los escáneres de códigos de barras, optar por consumirlos.

        return isInterrupt;
    }
```
### 6. Otros tratamientos
El proyecto requiere un escáner de códigos de barras externo, que tiene varios modos de funcionamiento:
1. Pulsación corta para iniciar el escaneo, soltar para detenerlo.
2. Pulsación corta para iniciar el escaneo continuo.
3. Activación por sensor, se detiene tras un tiempo de espera (este es el modo que se utilizará en el proyecto).

La razón por la que describo esto es que puede dar lugar a errores de manejo en interfaces no relacionadas, por ejemplo, en la interfaz x, donde escaneamos un código. Si no se gestiona, KEYCODE_ENTER responderá a un evento de clic en un botón de esa interfaz, lo que causará interferencias. Por lo tanto, necesitamos gestionar los eventos de entrada del escáner de códigos en la clase base BaseActivity de este proyecto. Actualmente, la estrategia de tratamiento que pretendo utilizar es que BaseActivity intercepte completamente los eventos del escáner de códigos de barras y que la interfaz que se necesite se abra por sí misma. El tratamiento aquí se considera un tratamiento de encapsulación, por lo que no lo describiré en detalle. Para más información, consulte el código de demostración.
### 7. Ejemplos y referencias
#### Nota:
La función AccessibilityService debe activarse manualmente en: Ajustes -> Accesibilidad -> Servicios. Se requiere formación humana, pero la interacción no es lo suficientemente intuitiva, por lo que se ha descartado.

#### Referencia:
[1]、https://stackoverflow.com/questions/11349542/handle-barcode-scanner-value-via-android-device
[2]、https://blog.csdn.net/csdnno/article/details/79639426

#### Demostración del proyecto
Código: https://github.com/lckj686/BarcodeScannerGunMaster
