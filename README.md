# Telegram Android – Accessibility Audit (Educational Fork)

> ⚠️ This repository is an educational fork of the official Telegram Android client.
>  
> It was created solely for academic accessibility analysis purposes.
>  
> This project is **not affiliated with or endorsed by Telegram** and does not distribute modified builds.
>  
> All rights belong to the original Telegram developers.

---

## Статус

Репозиторий используется исключительно в образовательных целях.
Изменения носят исследовательский характер.
Сборки для распространения не публикуются.

## Оригинальный проект

Официальный репозиторий Telegram Android:
https://github.com/DrKLO/Telegram

Все авторские права принадлежат разработчикам Telegram.

## Цель проекта

Данный репозиторий представляет собой форк официального Android-клиента Telegram, созданный в рамках учебного задания по анализу доступности пользовательских интерфейсов.

Цель работы — проведение аудита доступности интерфейса приложения для экранных дикторов (TalkBack), выявление проблем и формулировка рекомендаций по их устранению.

В рамках задания был выполнен:

- fork официального Android-клиента Telegram;

- анализ проблем доступности интерфейса;

- аудит взаимодействия с экранным диктором (screen reader);

- предложение исправлений с примерами кода;

**Объект исследования**: 
Android-клиент Telegram (проект **TMessagesProj**, т.е. основной модуль Android-клиента)

# Средства и методология анализа

### Используемые инструменты

В ходе аудита использовались:

- **TalkBack** (Android Screen Reader);

- **Accessibility Scanner (Google)**;

- Android Studio:
  
  - Layout Inspector;
  
  - Code Search;
  
  - Lint (проверки на Accessibility);

- Ручной анализ исходного кода

### Методика тестирования

Проверялись следующие пользовательские сценарии:

1. Навигация по списку чатов;

2. Открытие диалога;

3. Ввод и отправка сообщения;

4. Работа с голосовыми/видео сообщениями;

5. Работа с участниками групп;

6. Работа с кнопками;

7. Экран ввода пароля;

Проверялись критерии:

- Наличие описания для TalkBack;

- Роль элемента кнопок и кнопок с изображениями;

- Порядок фокуса;

- Озвучивание изменений состояния;

- Размеры некоторых элементов;

- Группировка логических блоков;

---

## Архитектура UI проекта

Telegram Android активно использует:

- кастомные представления (**View**);

- программное создание интерфейса;

- собственные контейнеры (Cells, Components);

Это увеличивает риск ошибок доступности, поскольку:

- элементы создаются вручную;

- доступность не всегда автоматически наследуется;

- требуется ручная настройка перехватчика поведения доступности у View и порядка обхода элементов скринридером;

## Найденные проблемы и способы решения

1. Кнопка закрытия **Reply panel** в чате. Clickable ImageView без **contentDescription**. TalkBack не сможет озвучить название кнопки.

```java
     replyCloseImageView.setColorFilter(new PorterDuffColorFilter(getThemedColor(Theme.key_glass_defaultIcon), PorterDuff.Mode.MULTIPLY));
     replyCloseImageView.setImageResource(R.drawable.input_clear);
     replyCloseImageView.setScaleType(ImageView.ScaleType.CENTER);
     replyCloseImageView.setBackgroundDrawable(Theme.createSelectorDrawable(getThemedColor(Theme.key_listSelector), 1, AndroidUtilities.dp(19)));
     chatActivityEnterTopView.addView(replyCloseImageView, LayoutHelper.createFrame(52, 46, Gravity.RIGHT | Gravity.TOP, 0, 0.5f, 0, 0));
     replyCloseImageView.setOnClickListener(v -> {
         messageSuggestionParams = null;
         if (fieldPanelShown == 2) {
             replyingQuote = null;
             replyingMessageObject = null;
             if (messagePreviewParams != null) {
                 messagePreviewParams.updateReply(null, null, dialog_id, null);
             }
             fallbackFieldPanel();
         } else if (fieldPanelShown == 3) {
             openAnotherForward();
         } else if (fieldPanelShown == 4) {
             foundWebPage = null;
             if (messagePreviewParams != null) {
                 messagePreviewParams.updateLink(currentAccount, null, null, replyingMessageObject == threadMessageObject ? null : replyingMessageObject, replyingQuote, editingMessageObject);
             }
             chatActivityEnterView.setWebPage(null, false);
             editResetMediaManual();
             fallbackFieldPanel();
         } else {
             if (ChatObject.isForum(currentChat) && !isTopic && replyingMessageObject != null) {
                 long topicId = MessageObject.getTopicId(currentAccount, replyingMessageObject.messageOwner, true);
                 if (topicId != 0) {
                     getMediaDataController().cleanDraft(dialog_id, topicId, false);
                 }
             }
             showFieldPanel(false, null, null, null, null, true, 0, null, true, 0, true);
         }
     });
```

**Файл**: [ChatActivity.java](TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java)

**Проблема:** элемент кликабельный, но у элемента **нет описания**, TalkBack не прочитает, пользователь не понимает назначение.

**Решение**: добавить описание элементу

```java
replyCloseImageView.setContentDescription(
    LocaleController.getString(R.string.Close)
);
replyCloseImageView.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
```

2. Кнопка фонарика в **CameraScanActivity**:

```java
flashButton = new ImageView(context);
flashButton.setScaleType(ImageView.ScaleType.CENTER);
flashButton.setImageResource(R.drawable.qr_flashlight);
flashButton.setBackgroundDrawable(Theme.createCircleDrawable(dp(60), 0x22ffffff));
viewGroup.addView(flashButton);
flashButton.setOnClickListener(currentImage -> {
    if (cameraView == null) {
        return;
    }
    CameraSessionWrapper session = cameraView.getCameraSession();
    if (session != null) {
        ShapeDrawable shapeDrawable = (ShapeDrawable) flashButton.getBackground();
        if (flashAnimator != null) {
            flashAnimator.cancel();
            flashAnimator = null;
        }
        flashAnimator = new AnimatorSet();
        ObjectAnimator animator = ObjectAnimator.ofInt(shapeDrawable, AnimationProperties.SHAPE_DRAWABLE_ALPHA, flashButton.getTag() == null ? 0x44 : 0x22);
        animator.addUpdateListener(animation -> flashButton.invalidate());
        flashAnimator.playTogether(animator);
        flashAnimator.setDuration(200);
        flashAnimator.setInterpolator(CubicBezierInterpolator.DEFAULT);
        flashAnimator.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                flashAnimator = null;
            }
        });
        flashAnimator.start();
        if (flashButton.getTag() == null) {
            flashButton.setTag(1);
            session.setCurrentFlashMode(Camera.Parameters.FLASH_MODE_TORCH);
        } else {
            flashButton.setTag(null);
            session.setCurrentFlashMode(Camera.Parameters.FLASH_MODE_OFF);
        }
   }
});
```

**Файл**: [CameraScanActivity.java](TMessagesProj/src/main/java/org/telegram/ui/CameraScanActivity.java)

**Проблема:** отсутствует label у **flashButton** для TalkBack.

**Решение**: добавить label для flashButton:

```java
flashButton.setContentDescription(
    LocaleController.getString(R.string.Flashlight)
);
```

и добавить состояние:

```java
flashButton.setSelected(isFlashOn);
flashButton.setContentDescription(isFlashOn ? "Выключить фонарик" : "Включить фонарик");
```

3. Кнопка закрытия **actionModeCloseView** (CallLogActivity):

```java
actionModeCloseView = new ImageView(getContext());
actionModeCloseView.setScaleType(ImageView.ScaleType.CENTER);
actionModeCloseView.setImageDrawable(new BackDrawable(true));
actionModeCloseView.setColorFilter(new PorterDuffColorFilter(getThemedColor(Theme.key_actionBarActionModeDefaultIcon), PorterDuff.Mode.MULTIPLY));
actionModeCloseView.setBackground(Theme.createSelectorDrawable(getThemedColor(Theme.key_actionBarActionModeDefaultSelector)));
actionModeCloseView.setOnClickListener(v -> hideActionMode(true));
actionMode.addView(actionModeCloseView, LayoutHelper.createLinear(54, 54, Gravity.CENTER_VERTICAL));
actionModeViews.add(actionModeCloseView);
```

**Файл**: [CallLogActivity.java](TMessagesProj/src/main/java/org/telegram/ui/CallLogActivity.java)

**Проблема:** Кнопка закрытия “actionModeCloseView” без contentDescription. TalkBack не сможет озвучить название кнопки.

**Решение**: добавить contentDescription для actionModeCloseView:

```java
actionModeCloseView.setContentDescription(
    LocaleController.getString(R.string.Close)
);
```

4. Кнопка голос/видео-сообщения (**ChatActivityEnterView**):

```java
audioVideoSendButton.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_NO);
//        audioVideoSendButton.setFocusable(true);
//        audioVideoSendButton.setAccessibilityDelegate(mediaMessageButtonsDelegate);
```

**Файл**: [ChatActivityEnterView.java](TMessagesProj/src/main/java/org/telegram/ui/Components/ChatActivityEnterView.java)

**Проблема:** Данной кнопке выставили `IMPORTANT_FOR_ACCESSIBILITY_NO`, а полезная настройка accessibility delegate закомментирована. В таком случае  TalkBack может не фокусировать реальную кнопку или фокусировать “контейнер” без правильных ролей или давать непредсказуемое поведение (особенно при долгом удержании/жестах записи).

**Решение**: сделать audioVideoSendButton доступной:

```java
audioVideoSendButton.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
audioVideoSendButton.setFocusable(true);
audioVideoSendButton.setClickable(true);
audioVideoSendButton.setAccessibilityDelegate(mediaMessageButtonsDelegate);
```

5. Состояния и динамика UI не всегда озвучиваются корректно. Возьмём для примера фрагмент (микрофон mute/unmute):

```java
final boolean micMute = !serviceInstance.isMicMute();
if (accessibilityManager.isTouchExplorationEnabled()) {
    final String text = LocaleController.getString(micMute ? R.string.AccDescrVoipMicOff : R.string.AccDescrVoipMicOn);
    view.announceForAccessibility(text);
}
serviceInstance.setMicMute(micMute, false, true);
```

**Файл**: [VoIPFragment.java](TMessagesProj/src/main/java/org/telegram/ui/VoIPFragment.java)

**Проблема:** Нет озвучивания изменений. А любая кнопка, которая меняет режим/состояние (пример: “звук выкл/вкл”, “режим видео/голос”, “закрепить/открепить”, “фильтр включён”), должна озвучивать изменение.

**Решение**: реализовать озвучивание изменений:

```java
AccessibilityManager am = (AccessibilityManager) getContext().getSystemService(Context.ACCESSIBILITY_SERVICE);
if (am != null && am.isEnabled() && am.isTouchExplorationEnabled()) {
    // Вот тут необходимо озвучивать
    String a11yText = enabled
        ? LocaleController.getString(R.string.AccDescrEnabled)   // "Включено"
        : LocaleController.getString(R.string.AccDescrDisabled); // "Выключено"
    targetView.announceForAccessibility(a11yText);
}
```

6. Кастомные View не всегда имеют корректную “роль”. Возьмём для примера блок меню с кнопками:

```java
@Override
public void onInitializeAccessibilityNodeInfo(AccessibilityNodeInfo info) {
    super.onInitializeAccessibilityNodeInfo(info);
    if (iconView != null) {
        info.setClassName("android.widget.ImageButton");
    } else if (textView != null) {
        info.setClassName("android.widget.Button");
    }
}
```

**Файл**: [ActionBarMenuItem.java](TMessagesProj/src/main/java/org/telegram/ui/ActionBar/ActionBarMenuItem.java)

**Проблема:** Многие элементы UI, включая кастомные контейнеры/иконки TalkBack может озвучивать их как “View”, без роли и без правильных действий.

**Решение**: добавить элементам роль, описание и кликабельность:

```java
ViewCompat.setAccessibilityDelegate(targetView, new AccessibilityDelegateCompat() {
    @Override
    public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfoCompat info) {
        super.onInitializeAccessibilityNodeInfo(host, info);

        info.setClassName(android.widget.Button.class.getName()); // роль "кнопка"
        info.setClickable(true);

        CharSequence cd = host.getContentDescription();
        if (TextUtils.isEmpty(cd)) {
            info.setContentDescription(LocaleController.getString(R.string.OpenAttachments));
        }
    }

    @Override
    public boolean performAccessibilityAction(View host, int action, Bundle args) {
        // кастомные a11y-actions
        return super.performAccessibilityAction(host, action, args);
    }
});
```

Также пример, если нужно добавить действие типа “Удалить/Переименовать”:

```java
ViewCompat.replaceAccessibilityAction(
    targetView,
    AccessibilityNodeInfoCompat.AccessibilityActionCompat.ACTION_LONG_CLICK,
    LocaleController.getString(R.string.Delete),
    (view, args) -> { performDelete(); return true; }
);
```

7. Порядок фокуса и обхода элементов нестабилен.

В данном компоненте реализована явная настройка порядка фокуса через  setAccessibilityTraversalBefore().

```java
    private void setNextFocus(View view, @IdRes int nextId) {
        view.setNextFocusForwardId(nextId);
        if (Build.VERSION.SDK_INT >= 22) {
            view.setAccessibilityTraversalBefore(nextId);
        }
    }
```

**Файл**: [PasscodeView.java](TMessagesProj/src/main/java/org/telegram/ui/Components/PasscodeView.java)

**Проблема:** TalkBack, как бы “скачет” между полями/кнопками не по логике экрана (особенно на формах, диалогах, экранах настроек, поиске).

**Решение**: проставить traversal цепочку на ключевых элементах UI:

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP_MR1) {
    viewA.setAccessibilityTraversalBefore(viewB.getId());
    viewB.setAccessibilityTraversalAfter(viewA.getId());
}
```

Но есть ситуация, когда контейнер перехватывает фокус. Мы можем увидеть пример в файле [GroupCreateActivity.java](TMessagesProj/src/main/java/org/telegram/ui/GroupCreateActivity.java)

```java
contentView.setFocusableInTouchMode(true);
contentView.setDescendantFocusability(ViewGroup.FOCUS_BEFORE_DESCENDANTS);
```

И мы можем указать в зависимости от того, кто должен получать фокус первым:

```java
contentView.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
contentView.setFocusable(false);
contentView.setFocusableInTouchMode(false);
```

8. Из проблем доступности стоит упомянуть и о маленьких зонах нажатия у иконок и кнопок. Возьмём для примера место из файла, который реализует кастомную ячейку пользователя (аватар, имя, статус, действия) для списков.

```java
closeView = new ImageView(getContext());
closeView.setScaleType(ImageView.ScaleType.CENTER);
ScaleStateListAnimator.apply(closeView);
closeView.setImageResource(R.drawable.ic_close_white);
closeView.setColorFilter(new PorterDuffColorFilter(Theme.getColor(Theme.key_windowBackgroundWhiteGrayText3, resourcesProvider), PorterDuff.Mode.SRC_IN));
closeView.setBackground(Theme.createSelectorDrawable(Theme.getColor(Theme.key_listSelector, resourcesProvider), Theme.RIPPLE_MASK_CIRCLE_AUTO));
addView(closeView, LayoutHelper.createFrame(30, 30, (LocaleController.isRTL ? Gravity.LEFT : Gravity.RIGHT) | Gravity.CENTER_VERTICAL, LocaleController.isRTL ? 14 : 0, 0, LocaleController.isRTL ? 0 : 14, 0));
```

**Файл**: [UserCell.java](TMessagesProj/src/main/java/org/telegram/ui/Cells/UserCell.java)

**Проблема:** размер у кнопки закрытия окна 30×30 dp, что довольно мало для хорошей доступности.

**Решение**: увеличить размер **closeView** до 48dp:

```java
addView(closeView, LayoutHelper.createFrame(48, 48,
    (LocaleController.isRTL ? Gravity.LEFT : Gravity.RIGHT) | Gravity.CENTER_VERTICAL,
    LocaleController.isRTL ? 14 : 0, 0, LocaleController.isRTL ? 0 : 14, 0
));
```

## Выводы

В результате анализа выявлено:

- часть элементов реализована с учётом доступности (VoIPFragment, PasscodeView);

- значительная часть интерактивных иконок требует явного описания для TalkBack;

- есть элементы с уменьшенной зоной касания;

- активно используются кастомные **View**, что требует ручной настройки доступности;

## Заключение

Telegram Android частично учитывает требования доступности, однако из-за сложной кастомной архитектуры интерфейса остаются проблемные зоны, требующие ручной доработки.


