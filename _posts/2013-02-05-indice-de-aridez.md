---
layout: post
title: Índice de aridez
date: 2013-02-05 21:54
author: dnietolugilde
comments: true
categories: [Aridez, Bash, ETP, GRASS, Script, Scripts]
---
Llevo unos días preparando en GRASS una nueva variable bioclimática. Se trata de un índice de aridez, calculado en base a las precipitaciones y a la ETP (Thornwaite). En este script, hay un paso que me ha resultado interesante y es una ecuación para estimar la longitud del día en función del día del año y de la latitud. Dejo aquí el script completo por si a alguien le interesa. También dejo un fichero de clasificación que hace falta para un paso intermedio del script.

Descarga directa: <a title="Script para calcular índice de aridez" href="http://wdb.ugr.es/~dinilu/scripts/AridityIndex.sh" target="_blank">AridityIndex.sh</a>

Fichero de reclasificación: <a title="Fichero de clasificación usado en el script" href="http://wdb.ugr.es/~dinilu/scripts/ClassificationRules.txt" target="_blank">ClassificationRules.txt</a>

<a title="Fichero de clasificación usado en el script" href="http://wdb.ugr.es/~dinilu/scripts/ClassificationRules.txt" target="_blank"><!--more--></a>
<pre>#!/bin/bash

################################
# Cálculo de índice de aridez

# IA = ETPthonwaite / Panual (mensual o anual)

# ETP thornwaite = e · L (mensuales)

# e = 16⋅(10⋅tm/I)^a (mensuales)

# tm = temperature media mensual (mensuales)

# I = Σi ; i= (tm/5)^1,514 (I anual; i mensuales)

# a = 0,000000675⋅I^3 - 0,0000771⋅I^2 + 0,01792⋅I + 0,49239 (anual)

# L = Nd/30 ⋅ Nh/12 (mensuales) ; Nd = número de días del mes ; Nh = Número de horas de luz 

####################
# Script:

# Especificamos el nombre de las variables de temperatura media mensual y Precipitación Anual
tm=tasmed_6k_CCSM.
Panual=bio12_6k_CCSM_30s

# Calculamos el parámetro "i" para cada mes a partir de los datos de temp media mensual
for j in $(seq 1 12)
do 
    r.mapcalculator amap=${tm}${j} 'formula=if(A,((A/10)/5)^1.514,0,0)' outfile=i_${j} --overwrite help=-
done

# Obtenemos el parámetro "I" como la suma de los "i" mensuales
r.series input="g.mlist pattern=i_* sep=," output=I method=sum --overwrite

# Borramos los mapas mensuales "i"
g.mremove -f rast=i_* 

# Calculamos el parámetro "a"
r.mapcalculator amap=I 'formula=(0.000000675*(A^3))-(0.000077*(A^2))+(0.01792*(A))+0.49239' outfile=a --overwrite help=-

# Calculamos un valor parcial (e1) de "e"
for j in $(seq 1 12)
do 
    r.mapcalculator amap=${tm}${j} bmap=I cmap=a 'formula=16*(((10*(A/10))/B)^C)' outfile=e1_${j} --overwrite help=-
done

# Borramos los mapas de "a" y "I"
g.mremove -f rast=a,I

# Calculamos un valor parcial (e2) de "e" que depende de la longitud del mes (número de días)
for j in $(seq 1 12)
do 
    r.reclass input=${tm}${j} output=e2_${j} rules=/media/diego/LaCie/Cartografia/Scripts/GRASS/Topillo_Paleoclim_Paleobioclim_WC-PMIP2/28-ClassificationRules.txt --overwrite

    if [[ ${j} == 1 || ${j} == 3 || ${j} == 5 || ${j} == 7 || ${j} == 8 || ${j} == 10 || ${j} == 12 ]];
        then
        r.mapcalculator amap=e2_${j} 'formula=(A/10)*31' outfile=e2_${j} --overwrite help=-
    fi
    if [[ ${j} == 4 || ${j} == 6 || ${j} == 9 || ${j} == 11 ]];
        then
        r.mapcalculator amap=e2_${j} 'formula=(A/10)*30' outfile=e2_${j} --overwrite help=-
    fi
    if [[ ${j} == 2 ]];
        then
        r.mapcalculator amap=e2_${j} 'formula=(A/10)*28' outfile=e2_${j} --overwrite help=-
    fi
done

# Calculamos "e"
for j in $(seq 1 12)
do 
    r.mapcalculator amap=${tm}${j} 'formula=A-265' outfile=${tm}_tmp_${j} --overwrite help=-
    r.mapcalculator amap=${tm}_tmp_${j} bmap=e1_${j} cmap=e2_${j} 'formula=if(A, C, C, B)' outfile=e_${j} --overwrite help=-
done

# Borramos los ficheros temporales creados "e1", "e2" y "tempMedia_tmp"
g.mremove -f rast=${tm}_tmp_*
g.mremove -f rast=e1_* 
g.mremove -f rast=e2_* 

# Ahora vamos a calcular la longitud del día media mensual. Usamos la siguiente referencia:
#Ecological Modeling, volume 80 (1995) pp. 87-95, "A Model Comparison for Daylength as a Function of Latitude and Day of the Year."

# Iniciamos obteniendo un mapa de latitud
r.mapcalculator amap=${tm}_1 'formula=y()' outfile=Latitud --overwrite help=-

# Obtenemos el parámetro "P", que depende del día del año, para cada latitud en el área de estudio.
declare -a P
for j in $(seq 1 365)
do
    tmp1=$(echo "scale=12; (0.0086*(${j} - 186))" | bc -l)
    tmp2=$(echo "scale=12; 0.9671396*(s(${tmp1})/c(${tmp1}))" | bc -l)
    tmp3=$(echo "scale=12; 0.2163108+(2*a(${tmp2}))" | bc -l)
    tmp4=$(echo "scale=12; 0.39795*c(${tmp3})" | bc -l)
    tmp5=$(echo "scale=12; a((${tmp4})/(sqrt(1 - (${tmp4}^2))))" | bc -l)

    i=$((j-1))
    P[$i]=echo "${tmp5}"
done

# Calculamos la longitud del día en función del día del año y de la latitud
for j in $(seq 1 365)
do
    i=${P[$((j-1))]}
    h=$(echo "scale=8; ${i}*180/3.1415925684" | bc)
    r.mapcalc "lengthday_${j}" = "24 - ( (24/3.1415925684) * (3.1415925684/180) *acos( ( sin(0.8333) + ( sin(Latitud) * sin(${h}) ) ) / ( cos(Latitud) * cos(${h}) ) ) )"
done

# Calculamos la longitud media del día para cada mes del año
r=echo lengthday_{1..31}
r=${r// /,}
r.series input=${r} output=D_1 method=average --overwrite 
r=echo lengthday_{32..59}
r=${r// /,}
r.series input=${r} output=D_2 method=average --overwrite 
r=echo lengthday_{60..90}
r=${r// /,}
r.series input=${r} output=D_3 method=average --overwrite 
r=echo lengthday_{91..120}
r=${r// /,}
r.series input=${r} output=D_4 method=average --overwrite 
r=echo lengthday_{121..151}
r=${r// /,}
r.series input=${r} output=D_5 method=average --overwrite 
r=echo lengthday_{152..181}
r=${r// /,}
r.series input=${r} output=D_6 method=average --overwrite 
r=echo lengthday_{182..212}
r=${r// /,}
r.series input=${r} output=D_7 method=average --overwrite 
r=echo lengthday_{213..243}
r=${r// /,}
r.series input=${r} output=D_8 method=average --overwrite 
r=echo lengthday_{244..273}
r=${r// /,}
r.series input=${r} output=D_9 method=average --overwrite 
r=echo lengthday_{274..304}
r=${r// /,}
r.series input=${r} output=D_10 method=average --overwrite 
r=echo lengthday_{305..334}
r=${r// /,}
r.series input=${r} output=D_11 method=average --overwrite 
r=echo lengthday_{335..365}
r=${r// /,}
r.series input=${r} output=D_12 method=average --overwrite 

# Borramos los datos mensuales para liberar espacio en el disco duro
g.mremove -f rast=lengthday_* 

# Calculamos el parámetro L en función de la duración del mes (número de días) y de la longitud media de los días (número de horas de luz)
r.mapcalc "L_1" = "(31/30)*(D_1/12)"
r.mapcalc "L_2" = "(28/30)*(D_2/12)"
r.mapcalc "L_3" = "(31/30)*(D_3/12)"
r.mapcalc "L_4" = "(30/30)*(D_4/12)"
r.mapcalc "L_5" = "(31/30)*(D_5/12)"
r.mapcalc "L_6" = "(30/30)*(D_6/12)"
r.mapcalc "L_7" = "(31/30)*(D_7/12)"
r.mapcalc "L_8" = "(31/30)*(D_8/12)"
r.mapcalc "L_9" = "(30/30)*(D_9/12)"
r.mapcalc "L_10" = "(31/30)*(D_10/12)"
r.mapcalc "L_11" = "(30/30)*(D_11/12)"
r.mapcalc "L_12" = "(31/30)*(D_12/12)"

# Borramos los datos de duración del día media para liberar espacio
g.mremove -f rast=D_*

# Calculamos la ETP mensual
for j in $(seq 1 12)
do
    r.mapcalc "ETP_${j}" = "e_${j} * L_${j}"
done

#Borramos los datos de "e" y "L" mensuales
g.mremove -f rast=e_*
g.mremove -f rast=L_*

# Calculamos la ETP media anual
r.series input=g.mlist pattern=ETP_* sep=, output=ETP method=sum --overwrite

# Borramos los datos de ETP mensuales
g.mremove -f rast=ETP_*

# Calculamos el índice de aridez
r.mapcalc "AridityIndex" = "(${Panual} / ETP)"

# Exportamos los ficheros en formato ascii
r.out.arc input=AridityIndex output=AridityIndex.asc dp=4
r.out.arc input=ETP output=ETP.asc dp=4</pre>
