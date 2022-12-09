*Данный урок является продолжением урока [Материалы](https://github.com/urho3d-learn/materials). Если вы с ним ещё не ознакомились, то начните с него.*

---

# Постэффекты (WIP)

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



---

*Старая версия демки: <https://github.com/1vanK/Urho3DHabrahabr06>.*

