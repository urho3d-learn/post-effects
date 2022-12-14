*Данный урок является продолжением урока [Материалы](https://github.com/urho3d-learn/materials). Если вы с ним ещё не ознакомились, то начните с него.*

---

# Постэффекты

Продолжаем разбираться в графической подсистеме Urho3D. На этот раз поговорим об эффектах постобработки. В комплект движка входит множество
уже готовых эффектов, и один из них (Bloom) мы даже использовали в [прошлом уроке](https://github.com/urho3d-learn/materials).
Но ни один движок не способен удовлетворить все нужды любого разработчика, поэтому будет полезно научиться создавать свои
собственные эффекты. В качестве примера я решил выбрать эффект просвечивания персонажа через стены,
который нередко используется в стратегиях и РПГ.

![](images/screen.png)

## Идея

Простыми словами, постэффект представляет собой прямоугольный полигон, закрывающий весь экран. И наша задача - раскрасить этот полигон.

<details>
<summary><b>Примечание</b></summary>
Сразу оговорюсь, что я не ставил перед собой цель реализовать эффект самым оптимальным методом.
В первую очередь я хотел показать, как сделать это с помощью постпроцессинга максимально простым и понятным способом.
</details>

Итак, нам нужно:
1) Отрендерить сцену
2) Получить маску невидимой части персонажа
3) Окрасить маску в какой-нибудь цвет и наложить ее на рендер сцены

Чтобы получить маску невидимой части можно:
1) Повторно отрендерить персонажа с включенным тестом глубины, используя простой шейдер, выводящий белый пиксель
   (получаем таким образом чёрно-белую текстуру видимой части персонажа)
2) Повторно отрендерить персонажа, используя тот же шейдер, но уже игнорируя буфер глубины (получаем таким образом чёрно-белую маску всего персонажа)
3) Если в каком-то месте <ins>полная маска</ins> белая, а <ins>маска видимой части</ins> чёрная, то это и есть искомая невидимая часть персонажа

Итого нам нужно получить и скомбинировать три текстуры:

![](images/idea.gif)

## Реализация

Батники для скачивания и компиляции рассматриваемой версии движка находятся в папке [engine](engine).

Готовая демка находится в папке [demo](demo).

## Загрузка рендерпасов

В [прошлом уроке](https://github.com/urho3d-learn/materials) мы рассматривали рендерпасы как способ задания очередности проходов в материалах (раздел `Процесс рендеринга`). Но рендерпасы выполняют также и [другие функции](https://urho3d-doxygen.github.io/1_9_0_tutors/_render_paths.html).

Стандартные рендерпасы находятся в папке `CoreData/RenderPaths`. По умолчанию используется `Forward.xml`. Сменить рендерпас можно разными способами:

* Вызвать функцию `Renderer::SetDefaultRenderPath()` **перед** созданием вьюпорта. При этом последующие создаваемые вьюпорты будут использовать указанный рендерпас. Данный метод и [используется](demo/MyData/Scripts/Main.as) в демке.
* Указать рендерпас в параметрах движка (с помощью [параметров командной строки](https://urho3d-doxygen.github.io/1_9_0_tutors/_running.html) при запуске приложения или через `engineParameters_` в тексте программы). При этом вызывается всё та же функция `Renderer::SetDefaultRenderPath()`.
* Использовать функцию `Viewport::SetRenderPath()` **после** создания вьюпорта.
* В редакторе рендерпас можно указать в окне `View` > `Editor Settings`.

Рендерпасы можно не только загружать из файлов, но и динамически изменять в процессе работы приложения. Например, когда вы применяете какой-нибудь постэффект из папки `Data/PostProcess`, происходит ни что иное, как **добавление** команд в текущий рендерпас. Иными словами вы можете просто скопировать содержимое какого-то файла (или файлов) из `Data/PostProcess` в конец какого-то файла из `CoreData/RenderPaths`.

[Scripts/Main.as](demo/MyData/Scripts/Main.as):

```
void Start()
{
    ...
    renderer.SetDefaultRenderPath(cache.GetResource("XMLFile", "RenderPaths/MyForward.xml"));
    Viewport@ viewport = Viewport(scene_, cameraNode.GetComponent("Camera"));
    viewport.renderPath.Append(cache.GetResource("XMLFile", "PostProcess/FXAA3.xml"));
    renderer.viewports[0] = viewport;
}
```

Здесь происходит загрузка рендерпаса [MyForward.xml](demo/MyData/RenderPaths/MyForward.xml) (который основан на [Forward.xml](demo/CoreData/RenderPaths/Forward.xml)), а затем к нему добавляется эффект полноэкранного сглаживания [FXAA3.xml](demo/Data/PostProcess/FXAA3.xml).

## Рендертаргеты

Рендерпасы состоят из рендертаргетов (rendertarget) и команд (command).

Рендератергеты — это, грубо говоря, текстуры, которые можно передавать в команды как входные данные, либо наоборот, команды могут выводить результат своей работы в них.

[RenderPaths/MyForward.xml](demo/MyData/RenderPaths/MyForward.xml):

```
<renderpath>
    <rendertarget name="visiblemask" tag="WallHack" sizedivisor="1 1" format="a" />
    <rendertarget name="fullmask" tag="WallHack" sizedivisor="1 1" format="a" />
    ...
</renderpath>
```

Здесь объявляется два рендертаргета для масок (visiblemask — маска видимой части персонажа и fullmask — маска всего персонажа).

Параметр `name` определяет имя рендертаргета, по которому к нему можно обращаться.

Параметр `tag` позволяет определить рендертаргеты и команды в какую-то группу, которую можно будет динамически включать и отключать в игре при помощи функций `RenderPath::SetEnabled()` и `RenderPath::ToggleEnabled()`. Обратите внимание, что все стандартные постэффекты имеют собственный тег. Таким образом можно, например, включать размытие экрана только при открытии меню. Ну а в нашей демке по нажатию пробела производится переключение эффекта просвечивания.

[Scripts/Main.as](demo/MyData/Scripts/Main.as):

```
void HandleUpdate(StringHash eventType, VariantMap& eventData)
{
    ...
    if (input.keyPress[KEY_SPACE])
        renderer.viewports[0].renderPath.ToggleEnabled("WallHack");
    ...
}
```

Параметр `sizedivisor` позволяет создать рендертаргет с размером, отличным от размера вьюпорта. В нашей демке размер масок идентичен размеру окна (так как вьюпорт занимает все окно игры). Но вот, например, в постэффекте Bloom.xml размер рендертаргета в 4 раза меньше размера вьюпорта в целях производительности (для накладываемого свечения высокое разрешение не требуется).

Параметр `format` определяет, собственно, формат рендертаргета. Наиболее часто используются форматы `rgb` и `rgba`, но в нашем случае для хранения масок достаточно одного канала, поэтому используется одноканальный формат `a`. Тут есть нюанс. В OpenGL 2 формату `a` соответствует GL_ALPHA (а значит нужно работать с каналом alpha), а в OpenGL 3 — GL_R8 (нужно работать с каналом red). Мы к этому еще вернемся при рассмотрении шейдеров.

Обратите внимание, что рендертаргеты не обязательно должны объявляться в начале файла. Они могут находиться в любом месте, даже в конце файла. Гарантируется, что все рендертаргеты будут созданы перед выполнением команд.


И сразу же о первой команде, которая нам понадобится — `clear`.


[RenderPaths/MyForward.xml](demo/MyData/RenderPaths/MyForward.xml):

```
<renderpath>
    ...
    <command type="clear" tag="WallHack" color="0 0 0 0" output="visiblemask" />
    <command type="clear" tag="WallHack" color="0 0 0 0" output="fullmask" />
    ...
</renderpath>
```

Здесь все просто — рендертаргеты закрашиваются чёрным цветом. Так как на каждом кадре белая маска персонажа прорисовывается в новом положении, то, если забыть очистку, получится белый шлейф.

Если параметр `output` отсутствует, то очищается вьюпорт. Иногда вы можете захотеть намеренно опустить очистку, например если вам нужен прозрачный фон вьюпорта.

## Команда scenepass

Это те самые проходы рендера, которые были упомянуты в [прошлом уроке](https://github.com/urho3d-learn/materials).

[RenderPaths/MyForward.xml](demo/MyData/RenderPaths/MyForward.xml):

```
<renderpath>
    ...
    <command type="scenepass" tag="WallHack" pass="visiblemask" output="visiblemask" />
    <command type="scenepass" tag="WallHack" pass="fullmask" output="fullmask" />
    ...
</renderpath>
```

Тут добавляются два прохода. Чтобы проходы были выполнены, они должны иметься также в [технике](demo/MyData/Techniques/DiffNormalWallHack.xml), которую использует [материал](demo/MyData/Materials/Mutant.xml) нашего персонажа:

[Techniques/DiffNormalWallHack.xml](demo/MyData/Techniques/DiffNormalWallHack.xml):

```
<technique ...>
    ...
    <pass name="visiblemask" vs="Mask" ps="Mask" depthwrite="false" depthtest="equal"  psexcludes="PACKEDNORMAL" />
    <pass name="fullmask" vs="Mask" ps="Mask" depthwrite="false" depthtest="always" psexcludes="PACKEDNORMAL" />
</technique>
```

Напомню, что объявление проходов в рендерпасе определяет `порядок` этих проходов, а проходы в технике определяют конкретные `шейдеры`, которые будут использоваться. В данном случае это минимально возможный шейдер [Mask](demo/MyData/Shaders/GLSL/Mask.glsl), выводящий белый пиксель.

К моменту проходов `visiblemask` и `fullmask` буфер глубины уже заполнен. Нам не нужно туда ничего писать, а только использовать его. Поэтому у обоих проходов параметр `depthwrite` выставлен в `false`. Однако параметр `depthtest` различен. При значении `depthtest="always"` буфер глубины игнорируется, и рисуется полная маска персонажа. При значении `depthtest="equal"` тест глубины будет пройден, только когда значение в Z-буфере совпадает с глубиной выводимой геометрии, то есть когда та же самая геометрия рендерится повторно.

## Команда quad

Именно эта команда и выводит закрывающий экран прямоугольный полигон, предназначенный для реализации постэффектов (так называемый screen quad).

[RenderPaths/MyForward.xml](demo/MyData/RenderPaths/MyForward.xml):

```
<renderpath>
    ...
    <command type="quad" tag="WallHack" vs="WallHack" ps="WallHack" output="viewport">
        <texture unit="diffuse" name="viewport" />
        <texture unit="normal" name="fullmask" />
        <texture unit="specular" name="visiblemask" />
    </command>
</renderpath>
```

Здесь для отрисовки квада используется шейдер WallHack, и на вход этого шейдера передаются 3 текстуры (viewport — отрендеренная сцена, fullmask — полная маска персонажа и visiblemask — маска видимой части персонажа). Пусть вас не путают названия текстурных юнитов (diffuse, normal и specular). Вы вольны передавать через них что угодно, и использовать в своих шейдерах как угодно.

Шейдер [WallHack](demo/MyData/Shaders/GLSL/WallHack.glsl) очень прост.

1\) Получаем цвет текселя отрендеренной сцены (напомню, что мы в [рендерпасе](demo/MyData/RenderPaths/MyForward.xml) передали рендер сцены через текстурный юнит diffuse):

[GLSL/WallHack.glsl](demo/MyData/Shaders/GLSL/WallHack.glsl):

```
void PS()
{
    vec3 viewport = texture2D(sDiffMap, vTexCoord).rgb;
    ...
}
```

2\) Получаем обе маски:


[GLSL/WallHack.glsl](demo/MyData/Shaders/GLSL/WallHack.glsl):

```
void PS()
{
    ...
    #ifdef GL3
        float fullmask = texture2D(sNormalMap, vTexCoord).r;
        float visiblemask = texture2D(sSpecMap, vTexCoord).r;
    #else
        float fullmask = texture2D(sNormalMap, vTexCoord).a;
        float visiblemask = texture2D(sSpecMap, vTexCoord).a;
    #endif
    ...
}
```

Возвращаясь к упомянутому выше нюансу, мы видим, что в зависимости от версии OpenGL при работе с одноканальной текстурой мы используем разные каналы.

<details>
<summary><b>Source/Urho3D/Graphics/OpenGL/OGLGraphics.cpp</b></summary>
<pre><code>unsigned Graphics::GetAlphaFormat()
{
#ifndef GL_ES_VERSION_2_0
    // Alpha format is deprecated on OpenGL 3+
    if (gl3Support)
        return GL_R8;
#endif
    return GL_ALPHA;
}</pre></code>
</details>

Если ваше оборудование поддерживает OpenGL 3, то по умолчанию используется эта версия. В целях тестирования вы можете заставить движок использовать OpenGL 2 с помощью параметра командной строки `-gl2`.

3\) И наконец все три текстуры комбинируются:

[GLSL/WallHack.glsl](demo/MyData/Shaders/GLSL/WallHack.glsl):

```
void PS()
{
    ...
    if (fullmask == 0.0 || visiblemask > 0.0)
        gl_FragColor = vec4(viewport, 1.0);
    else   
        gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

Если в данном месте <ins>полная маска</ins> чёрная (то есть фрагмент не принадлежит персонажу), то выводим тексель отрендеренной сцены. Иначе, если <ins>видимая маска</ins> не чёрная (то есть фрагмент принадлежит видимой части персонажа), то тоже выводим отрендеренную сцену (видимую часть персонажа во всей красе, с картами нормалей и прочим). Иначе остается третий и последний вариант — фрагмент принадлежит невидимой части персонажа. Вот его то и рисуем красным цветом.

## Вывод промежуточных текстур

Иногда в целях отладки вы можете захотеть вывести промежуточные рендертаргеты на экран. Для этого удобно использовать стандартный шейдер [CopyFramebuffer](demo/CoreData/Shaders/GLSL/CopyFramebuffer.glsl). Нужно просто добавить в конец рендерпаса:

```
<renderpath>
    ...
    <command type="quad" vs="CopyFramebuffer" ps="CopyFramebuffer" output="viewport">
        <texture unit="diffuse" name="ИМЯ НУЖНОЙ ТЕКСТУРЫ" />
    </command>
</renderpath>
```

Этот шейдер ожидает, что вы передадите ему нужную текстуру в формате `rgb(a)` через текстурный юнит diffuse. Для вывода нашей одноканальной маски он не подходит. Поэтому я слегка модифицировал этот шейдер, чтобы он ожидал на входе текстуру в формате `a` (смотрите шейдер [ShowATexture.glsl](demo/MyData/Shaders/GLSL/ShowATexture.glsl)). Именно с его помощью сделаны скриншоты для статьи.

```
<renderpath>
    ...
    <command type="quad" vs="ShowATexture" ps="ShowATexture" output="viewport">
        <texture unit="diffuse" name="fullmask" />
    </command>
</renderpath>
```

---

*Старая версия демки: <https://github.com/1vanK/Urho3DHabrahabr06>.*

