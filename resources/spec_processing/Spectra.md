## Обработка оптических спектров на примере РТТ-150/TFOSC на сервере

Создание краткой сводки по всем файлам

```bash
makelog
```

Создадим папку с названием нашей звезды стандарта и скопируем туда спектры данной звезды и файлы ламп **FeAr** и **Halogen**

```bash
mkdir BD+25d4655
cp `cat .log | grep 20200721 | grep '1024 200' | grep BD+25d4655 | awk '{print$3}'` BD+25d4655/
```

### Усредненный bias

Создадим папку со спектральными **bias** и скопируем их туда

```bash
mkdir biassp
cp `cat .log | grep '1024 200' | grep bias | grep 0.0000 | grep -v flat | awk '{print$3}'` biassp/
```

Проверим **bias** перед усреднением. 

```bash
cd biassp/
ls *.fit > biassp.list
gedit biassp.list & # для удаления "плохих" bias из списка
ds9 &
xs `cat biassp.list`
```

Перед тем как делать усреднение нужно исправить ключи в FITS Header к bias, так как у нас не записан правильный **GAIN**. Для этого выполним команду 

```bash
../bin/repair_tfosc_andor_keys *.fit
```

Усредним файлы **bias** по медиане. Для этого небходимо в **IRAF** открыть noao.imred.ccdred

```bash
epar zerocombine
input=@biassp.list
output=Biassp2.fits
combine=median
```

Ввести команду`:go`

Скопируем **Biassp2.fits** в папку **../Caldata**

```bash
cp Biassp2.fits ../Caldata/
```

Перейдем в папку **BD+25d4655** и исправвим ключи

```bash
../bin/repair_tfosc_andor_keys *.fit
```

Создадим списки:

	* спектров звезды spec.list 
	* спектров лампы **FeAr** fear.list
	* спектров лампы **Halogen** halogen.list

```bash
cat .log | grep 100.0 | awk '{print$3}' > spec.list
cat .log | grep Fe-Ar | awk '{print$3}' > fear.list
cat .log | grep Halogen | awk '{print$3}' > halogen.list
```

Вычтем из всх спектров усредненный **bias**

```bash
../bin/process *.fit | si
```

### Усреднение Halogen

В IRAF открываем noao.imred.ccdred

```bash
epar combine
input=@halogen.list
output=halogen
combine=average
```

Запускаем `:go`

### Усреднение FeAr

В IRAF открываем noao.imred.ccdred

```bash
epar combine
input=@fear.list
output=fear
combine=average
```



### Создание функции отклика

В IRAF twodspec.longslit 

```bash
epar response
calibrat=halogen
normaliz=halogen
response=nhalogen
```

Запускаем `:go`

### Деление спектра звезд на плоское поле

В IRAF imred.ccdred 

```bash
epar ccdproc
images=@spec.list
flatcor=yes
flat=nhalogen.fits
```

Запускаем `:go`

### Построение дисперсионной кривой

IRAF twodspec.longslit

```bash
epar identify
images=fear.fits
section=middle line
coordli=linelists$idhenear.dat
cradius=8
```

![](lines_rtt.png)





```bash
epar reidentify
cradius=8
coordli=linelists$idhenear.dat
referenc=fear
images=fear
```

```bash
epar fitcoords
images=fear
```

Удаляем плохие точки клавишами `d` `p`

`y` `y` `r`

`y``r``r`

![](fitcoords.png)

Всё замечательно вписалось

### Привязка спектра звезды по длинам волн

Трансформируем спектры по найденному в fitcoords решении

```bash
epar transform
input=@spec.list
output=tr//@spec.list
fitnames=fear
```

Запускаем `:go`

> В папке database два файла fcfear, idfear, хотя **fitnames**=fear 

### Очистка от космиков

```bash
 ~uskov/runTmp/do_cleancrspec trfosc025*.fit
```

### Совмещение спектров звезды для дальнейшего суммирования

IRAF в пакете iki команда alignspectra

```bash
epar alignspec
images=tr//@spec.list
refimage=trfosc0254.fit
prefix=rg
imreg=[160:670,85:145] # указываем бокс, по которому он будет совмещать спектры для дальнейшего складывания
```

### Комбинирование совмещенных спектров

В IRAF пакет imred.ccdred команда combine

```bash
combine input=rgtr//@spec.list output=bd+25d4655.fits
```



### Экстракция спектра

В IRAF пакет twodspec.apextract команда apall

```bash
apall input=bd+25d4655.fits output=bd+25d4655_end.fits
```

Клавиши управления

* `m` создать новую апертуру и центрировать
* `d` удалить апертуру
* `l` установить <u>левую</u> границу апертуры
* `u` установить <u>правую</u> границу апертуры
* `b` перейти в режим выбора фона. 
  * `z` удаляет область фона
  * `s` установить левую границу области, а потом правую
  * `q` выйти из режима фона
* `:o 3` изменить порядок аппроксимации фона 
* `f` аппроксимировать фон заново

В итоге получим изображение спектра звезды

![](spec_bd+25d.png)

### Перевод отсчетов в абсолютные потоки

После этого нужно создать файл стандарта звезды std.fits в пакете noao.onedspec

```bash
epar standard
input=bd+25d4655_end.fits 
output=std.fits 
star=mbd25d4655
caldir=./
```

Учет DQE и поглощения:

```bash
epar sensfunc 
standard=std.fits  
sensitiv=sens
```

Теперь переведем стандартный спектр в абсолютные потоки:

```bash
epar calibrate 
input=bd+25d4655_end.fits 
output=bd_flux.fits
sensiti=sens.0001.fits
```