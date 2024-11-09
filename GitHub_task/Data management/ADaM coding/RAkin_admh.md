---
title: "ADSL coding"
author: "RAkin Alexander"
date: "2024-11-08"
output: 
  html_document:
    keep_md: true
---

<style type="text/css">
body{
  font-family: Helvetica;
  font-size: 12pt;
}
/* Headers */
h1, h2{
  font-size: 16pt;
}
</style>



Первым делом подгрузим все нужные нам файлы. По условию задания необходимо посмотреть на колонку "Source / Derivation" файла Spec.



- ADSL.xlsx, 
-MH_MEDICALHISTORY.xlsx



``` r
adsl <- read.xlsx("ADaM-like/ADSL.xlsx")

mh_med <- read.xlsx("SDTM/MH_MEDICALHISTORY.xlsx")
```

Из ADSL "вытащим" следующие переменные:

ADSL.STUDYID
ADSL.USUBJID
ADSL.TRTP
ADSL.TRTPN
;


``` r
adsl_data <- 
  adsl %>% 
  select(STUDYID,SUBJID, USUBJID, TRTP, 
         TRTPN) %>% 
  # Согласно спецификации переменная TRTPN должна иметь Variable Type: integer

  mutate(TRTPN = as.integer( TRTPN ))
```

Из файла "MH_MEDICALHISTORY.xlsx" получим следующие переменные: 
MH.MHSEQ
MH.MHCAT
MH.MHTERM
MH.MHDECOD 
MH.MHBODSYS 
MH.MHSTDTC
MH.MHENDTC
MH.MHENRTPT






``` r
mh_med_data <-
  mh_med %>%
  ## Переменная SUBJID нужна для последущего обЪеденений датасетов
  select(SUBJID,
         MHSEQ,
         MHCAT,
         MHTERM,
         MHDECOD,
         MHBODSYS,
         MHSTDTC ,
         MHENDTC,
         MHENRTPT) %>%
  # По спецификации MH.MHCAT = 'Medical History'
  
  
  filter(MHCAT == 'Medical History') %>%
  # В соответсвии с пецификацией поставим правильные типы переменных
  mutate(MHSEQ = as.integer(MHSEQ)) %>%
  # Теперь сгруппируем датасет по переменной SUBJID и найдем  все заполненные  строчки MHTERM
  group_by(SUBJID) %>%
  
  
  
  
  
  
  filter(!is.na(MHTERM)) %>%

  # Добавим недостающие переменые
  # начнем с ASTDT. В спецификации написано
  #   "Change date display format for MH.MHSTDTC
  # If the day is missing then impute adding '-01' to the date,
  # otherwise if both the day and the month are missing then impute '-01-01' to the date,
  # otherwise if the date is missing completely then leave blank"
  
  # Можно использовать case_when()
  mutate(
    ASTDT = case_when(
      nchar(MHSTDTC) == 7 ~ format(as.Date(paste0(MHSTDTC, "-01", sep = ""),
                                           format = "%Y-%m-%d"), "%d.%m.%Y"),
      nchar(MHSTDTC) == 4 ~ format(as.Date(paste0(MHSTDTC, "-01-01", sep = ""),
                                           format = "%Y-%m-%d"), "%d.%m.%Y"),
      .default = format(as.Date(MHSTDTC, format = "%Y-%m-%d"), "%d.%m.%Y")),
    
    
    # А можно использовать функцию anytime::anydate()
    
    # mutate(ASTDT = format(anydate(MHSTDTC), "%d.%m.%Y"))
    
    # Теперь добавим переменную ASTDTF
    # "If start date is completely missing or missing the year then ASTDTF is ’Y’
    # Else if start date has month missing then ASTDTF is ’M’
    # Else if start date has day missing then ASTDTF is ’D’"
    ASTDTF = case_when(
      nchar(MHSTDTC) > 7 ~ "",
      nchar(MHSTDTC) == 7 ~ "D",
      nchar(MHSTDTC) == 4 ~ "M",
      .default = "Y"),
    
    # Теперь добавим переменную AENDT и  воспользуемся функцией anytime::anydate(), так меньше писать кода
    
    # "if MHENRTPT = 'ONGOING', then '', else different display format for MH.MHENDTC
        # If the day is missing then impute adding '-01' to the date,
        # otherwise if both the day and the month are missing then impute adding
        # '-01-01' to the date,
        # otherwise if the date is missing completely then leave blank"
    
    AENDT = case_when(
      MHENRTPT == 'ONGOING' ~ "",
      .default = format(anydate(MHENDTC), "%d.%m.%Y")),
    
    # Следующая переменная  AENDTF
        # "If end date is completely missing or missing the year then AENDTF is ’Y’
        # Else if end date has month missing then AENDTF is ’M’
        # Else if end date has day missing then AENDTF is ’D’."
    
    AENDTF = case_when(
      nchar(MHENDTC) > 7 ~ "",
      nchar(MHENDTC) == 7 ~ "D",
      nchar(MHENDTC) == 4 ~ "M",
      .default = "Y"),
    
    # Последняя переменная  MHENRF
    
    MHENRF = case_when(
      MHENRTPT == 'ONGOING' ~ "ONGOING")
    
  )
```


Сджойним нужные нам датафреймы:


``` r
ADMH_1 <- left_join(adsl_data, mh_med_data) %>%
  # Выберем нужные переменные
  select(
    STUDYID,
    USUBJID,
    TRTP,
    TRTPN,
    MHSEQ,
    MHCAT,
    MHTERM,
    MHDECOD,
    MHBODSYS,
    MHSTDTC,
    ASTDT,
    ASTDTF,
    MHENDTC,
    AENDT,
    AENDTF,
    MHENRTPT,
    MHENRF
  ) %>%
  # Уберем лишнюю строку
  filter(!is.na(MHTERM))
```

```
## Joining with `by = join_by(SUBJID)`
```

``` r
# Проверим формат переменных
str(ADMH_1)
```

```
## 'data.frame':	5 obs. of  17 variables:
##  $ STUDYID : chr  "XXXX1-PX-0X" "XXXX1-PX-0X" "XXXX1-PX-0X" "XXXX1-PX-0X" ...
##  $ USUBJID : chr  "XXXX1-PX-0X-001049" "XXXX1-PX-0X-001053" "XXXX1-PX-0X-001054" "XXXX1-PX-0X-001054" ...
##  $ TRTP    : chr  "XXXX0" "XXXX1" "XXXX1" "XXXX1" ...
##  $ TRTPN   : int  1 2 2 2 2
##  $ MHSEQ   : int  1 1 1 2 3
##  $ MHCAT   : chr  "Medical History" "Medical History" "Medical History" "Medical History" ...
##  $ MHTERM  : chr  "RADIOLOGICALLY ISOLATED SYNDROME" "LUNG SILICOSIS." "ESSENTIAL HYPERTENSION" "MENOPAUSE" ...
##  $ MHDECOD : chr  "RADIOLOGICALLY ISOLATED SYNDROME" "SILICOSIS" "ESSENTIAL HYPERTENSION" "MENOPAUSE" ...
##  $ MHBODSYS: chr  "NERVOUS SYSTEM DISORDERS" "INJURY, POISONING AND PROCEDURAL COMPLICATIONS" "VASCULAR DISORDERS" "SOCIAL CIRCUMSTANCES" ...
##  $ MHSTDTC : chr  "2022-12-14" "2014" "2020" NA ...
##  $ ASTDT   : chr  "14.12.2022" "01.01.2014" "01.01.2020" NA ...
##  $ ASTDTF  : chr  "" "M" "M" "Y" ...
##  $ MHENDTC : num  NA NA NA NA NA
##  $ AENDT   : chr  "" "" "" "" ...
##  $ AENDTF  : chr  "Y" "Y" "Y" "Y" ...
##  $ MHENRTPT: chr  "ONGOING" "ONGOING" "ONGOING" "ONGOING" ...
##  $ MHENRF  : chr  "ONGOING" "ONGOING" "ONGOING" "ONGOING" ...
```

``` r
# Заменим формат переменных

col_names_chr <- colnames(ADMH_1)

ADMH <- ADMH_1 %>%
  mutate(### конечно можно написать каждую переменную и изменить ее тип, но мы пойдем более сложным путем и пропишим ко всем колононкам, за исключением TRTPN, MHSEQ применим функцию  as.character
    ### Тут без подсказки  GPT не обощлось, как выбрать все значения вектора кроме некоторых
    
    across(all_of(col_names_chr[!col_names_chr %in% c("TRTPN", "MHSEQ")]),
           as.character))
```




``` r
write.xlsx(ADMH, "ADaM-like/ADMH.xlsx")
```

