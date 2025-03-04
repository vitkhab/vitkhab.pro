---
title: "Первый взгляд на Docker Scout"
date: 2025-03-03
publishdate: 2025-03-03
tags: ["docker"]
draft: false
comments: true
summary: "Первые впечатления от нового сканера безопасности встроенного в экосистему Docker"
---

# TL;DR
- Классно, что появился встроенный в экосистему сканер уязвимисотей.
- Если ничего не использовали, то можно начать с Docker Scout. Поможет обезопасить образы.
- Если использовали Trivy, то не вижу причин переходить на Docker Scout. У Trivy Больше функционала, качественнее базы.
- Чем более облегченный образ используете, тем безопаснее. Идеально использовать distroless.

# Пролог

Собрал для себя образ. И `docker build` предложил проверить его на уязвимостью новой утилитой  [Docker Scout][1]. А он как возьми, да найди сомнительные пакеты. «Не порядок» — подумал я и начал оптимизировать совсем не нуждающийся в этом образ. 

[1]: https://www.docker.com/products/docker-scout/

# Знакомство с инструментом

Я начал приключение с `Dockerfile` следующего содержания:
```dockerfile
FROM node:22

ARG SERVELESS_VERSION=3.40.0
ARG YC_PLUGIN_VERSION=1.7.25

# Install Serverless Framework globally and Yandex.Cloud plugin
RUN npm install -g \
    serverless@${SERVELESS_VERSION} \
    "@yandex-cloud/serverless-plugin@${YC_PLUGIN_VERSION}"

WORKDIR /app

CMD [ "serverless" ]
```

**Docker** порекомендовал собрать образ через команду `docker buildx build --provenance=true --sbom=true`. Но тут же свалился с ошибкой `trying to send message larger than max`.


{{< details summary="Лог ошибки" >}}
```shell
 => [linux/arm64] generating sbom using docker.io/docker/buildkit-syft-scanner:stable-1           12.1s
 => ERROR exporting to image                                                                      19.9s
 => => exporting layers                                                                           11.6s
 => => exporting manifest sha256:ddbffdb1f2f1dde3219cdd7e13de9c45f7968183a04d54065a55b8381f09ddd0  0.0s
 => => exporting config sha256:38eb1a60e1878cc720ec29ef2b55198479e2f2dc12a53206be13421b808c6ebb    0.0s
------
 > exporting to image:
------
ERROR: failed to solve: error writing data blob sha256:552c9fa6885ff2969db4a747c1194b74037d392aa234c708
f5b44ec4a5445229: failed to copy: failed to send write: trying to send message larger than max
(16955122 vs. 16777216): unknown
```
{{< /details >}}

Разберем команду по кусочкам:
- `buildx` — позволяет использовать расширенные функции BuildKit. Подробнее про [доступные команды и атрибуты][2].
- `--provenance=true` — дает больше информации о том, как был собран образ. Записывает информацию о шагах сборки, параметры и переменные окружения, timestamp'ы и так далее. Подробнее про [доступные значения флага][3] и о [проверке][4].
- `--sbom=true` — собирает информацию об используемых пакетах и библиотеках во время сборки. Но может вызвать ошибку на больших образах, поэтому я ею воспользуюсь позже. Подробнее про [доступные значения флага][5] и о [проверке][6].

[2]: https://docs.docker.com/reference/cli/docker/buildx/
[3]: https://docs.docker.com/reference/cli/docker/buildx/build/#provenance
[4]: https://docs.docker.com/build/metadata/attestations/slsa-provenance/
[5]: https://docs.docker.com/reference/cli/docker/buildx/build/#sbom
[6]: https://docs.docker.com/build/metadata/attestations/sbom/

Собираю образ. `quickview` рекомендует изменить образ на той же версии с тегом `-slim` и показывыает как сразу уменьшится количество угроз на порядок.

```shell
❯ docker buildx build --provenance=true --no-cache -t vitkhab/serverless:3.40.0-22 .

...

❯ docker scout quickview local://vitkhab/serverless:3.40.0-22
    ✓ Image stored for indexing
    ✓ Indexed 1372 packages
    ✓ Provenance obtained from attestation
    ✓ Pulled

  Target             │  local://vitkhab/serverless:3.40.0-22  │    0C     3H     8M   129L     4?   
    digest           │  271280766dd4                          │                                     
  Base image         │  node:22                               │    0C     3H     7M   129L     4?   
  Updated base image │  node:22-slim                          │    0C     0H     2M    23L          
                     │                                        │           -3     -5   -106     -4   
```

Команда `recommendations` дополнительно оценивает свежесть базового образа и в качестве рекомендаций предлагает альтернативу в виде образа с тегом `23-slim`, с более новой версией Node.js. Также проводит оценку по размеру образа и количеству установленных пакетов.

{{< details summary="Подробные рекомендации от `recommendations`" >}}
```shell
❯ docker scout recommendations local://vitkhab/serverless:3.40.0-22
    ✓ Image stored for indexing
    ✓ Indexed 1372 packages
    ✓ Provenance obtained from attestation
    ✓ Pulled

  Target   │  local://vitkhab/serverless:3.40.0-22
    digest │  271280766dd4                        

## Recommended fixes

  Base image is  node:22 

  Name            │  22                                                                        
  Digest          │  sha256:6f448489c19fb63b81d788d95a470293026b034d8d657c2f9e5a9d6955a8deec   
  Vulnerabilities │    0C     3H     7M   129L     4?                                          
  Pushed          │ 1 week ago                                                                 
  Size            │ 396 MB                                                                     
  Packages        │ 749                                                                        
  Runtime         │ 22.14.0                                                                    


Refresh base image
  Rebuild the image using a newer base image version. Updating this may result in breaking changes.

  ✓ This image version is up to date.


Change base image
  The list displays new recommended tags in descending order, where the top results are rated as most suitable.


              Tag              │                         Details                         │   Pushed   │          Vulnerabilities            
───────────────────────────────┼─────────────────────────────────────────────────────────┼────────────┼─────────────────────────────────────
   22-slim                     │ Benefits:                                               │ 1 week ago │    0C     0H     2M    23L          
  Minor runtime version update │ • Minor runtime version update                          │            │           -3     -5   -106     -4   
  Also known as:               │ • Image is smaller by 303 MB                            │            │                                     
  • 22.14.0-slim               │ • Image contains 423 fewer packages                     │            │                                     
  • 22.14-slim                 │ • Image introduces no new vulnerability but removes 114 │            │                                     
  • lts-slim                   │ • Tag is using slim variant                             │            │                                     
  • jod-slim                   │                                                         │            │                                     
  • 22-bookworm-slim           │ Image details:                                          │            │                                     
  • jod-bookworm-slim          │ • Size: 78 MB                                           │            │                                     
  • lts-bookworm-slim          │ • Runtime: 22.14.0                                      │            │                                     
  • 22.14-bookworm-slim        │                                                         │            │                                     
  • 22.14.0-bookworm-slim      │                                                         │            │                                     
                               │                                                         │            │                                     
                               │                                                         │            │                                     
   23-slim                     │ Benefits:                                               │ 1 week ago │    0C     0H     2M    23L          
  Tag is preferred tag         │ • Image is smaller by 302 MB                            │            │           -3     -5   -106     -4   
  Also known as:               │ • Image contains 423 fewer packages                     │            │                                     
  • 23.8.0-slim                │ • Tag is preferred tag                                  │            │                                     
  • 23.8-slim                  │ • Tag was pushed more recently                          │            │                                     
  • current-slim               │ • Image introduces no new vulnerability but removes 114 │            │                                     
  • slim                       │ • Tag is using slim variant                             │            │                                     
  • bookworm-slim              │                                                         │            │                                     
  • 23-bookworm-slim           │ Image details:                                          │            │                                     
  • 23.8-bookworm-slim         │ • Size: 80 MB                                           │            │                                     
  • 23.8.0-bookworm-slim       │ • Runtime: 22                                           │            │                                     
  • current-bookworm-slim      │                                                         │            │                                     
                               │                                                         │            │                                     
                               │                                                         │            │                                     
```
{{< /details >}}

Меняю базовый образ на `node:22-slim`, собираю и снова проверяю с помощью Docker Scout. Теперь остается только одна рекомендация - обновление базового образа на `node:23-slim`.

{{< details summary="Результаты сканирования образа на основе `22-slim`" >}}

```shell
❯ docker buildx build --sbom=true --provenance=true --no-cache -t vitkhab/serverless:3.40.0-22-slim .

...

❯ docker scout quickview local://vitkhab/serverless:3.40.0-22-slim
    ✓ Image stored for indexing
    ✓ SBOM obtained from attestation, 912 packages found
    ✓ Provenance obtained from attestation
    ✓ Pulled

  Target             │  local://vitkhab/serverless:3.40.0-22-slim  │    0C     0H     3M    23L   
    digest           │  6973d8cc1f2d                               │                              
  Base image         │  node:22-slim                               │    0C     0H     2M    23L   
  Updated base image │  node:23-slim                               │    0C     0H     2M    23L   
                     │                                             │                              

❯ docker scout recommendations local://vitkhab/serverless:3.40.0-22-slim
    ✓ SBOM of image already cached, 912 packages indexed

  Target   │  local://vitkhab/serverless:3.40.0-22-slim
    digest │  6973d8cc1f2d                        

## Recommended fixes

  Base image is  node:22-slim 

  Name            │  22-slim                                                                   
  Digest          │  sha256:0576f7912bf105a725a9f03fe84f0db764e15144c15a729752a3c3cbf7da5408   
  Vulnerabilities │    0C     0H     2M    23L                                                 
  Pushed          │ 1 week ago                                                                 
  Size            │ 78 MB                                                                      
  Packages        │ 326                                                                        
  Runtime         │ 22.14.0                                                                    


Refresh base image
  Rebuild the image using a newer base image version. Updating this may result in breaking changes.

  ✓ This image version is up to date.


Change base image
  The list displays new recommended tags in descending order, where the top results are rated as most suitable.


            Tag           │                  Details                   │   Pushed   │       Vulnerabilities        
──────────────────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────
   23-slim                │ Benefits:                                  │ 1 week ago │    0C     0H     2M    23L   
  Tag is preferred tag    │ • Same OS detected                         │            │                              
  Also known as:          │ • Tag is preferred tag                     │            │                              
  • 23.8.0-slim           │ • Tag was pushed more recently             │            │                              
  • 23.8-slim             │ • Image has similar size                   │            │                              
  • current-slim          │ • Image has same number of vulnerabilities │            │                              
  • slim                  │ • Image contains equal number of packages  │            │                              
  • bookworm-slim         │ • Tag is using slim variant                │            │                              
  • 23-bookworm-slim      │                                            │            │                              
  • 23.8-bookworm-slim    │ Image details:                             │            │                              
  • 23.8.0-bookworm-slim  │ • Size: 80 MB                              │            │                              
  • current-bookworm-slim │ • Runtime: 22                              │            │                              
                          │                                            │            │                              
                          │                                            │            │                              
                          │                                            │            │                              
```
{{< /details >}}

Продолжаю вестись на уговоры Docker Scout, обновляю базовый образ на `node:23-slim` и пересобираю. Теперь мне рекомендуют вернуться обратно на `node:22-slim` <nobr>¯\\\_(ツ)\_/¯</nobr> .

{{< details summary="Результаты сканирования образа на основе `23-slim`" >}}
```shell
❯ docker scout quickview local://vitkhab/serverless:3.40.0-23-slim
    ✓ Image stored for indexing
    ✓ SBOM obtained from attestation, 912 packages found
    ✓ Provenance obtained from attestation
    ✓ Pulled

  Target             │  local://vitkhab/serverless:3.40.0-23-slim  │    0C     0H     3M    23L   
    digest           │  7c0c3361b083                               │                              
  Base image         │  node:23-slim                               │    0C     0H     2M    23L   
  Updated base image │  node:22-slim                               │    0C     0H     2M    23L   
                     │                                             │                              

❯ docker scout recommendations local://vitkhab/serverless:3.40.0-23-slim                    
    ✓ SBOM of image already cached, 912 packages indexed

  Target   │  local://vitkhab/serverless:3.40.0-23-slim
    digest │  7c0c3361b083                        

## Recommended fixes

  Base image is  node:23-slim 

  Name            │  23-slim                                                                   
  Digest          │  sha256:2dbdc1134da545b2a24e30678676c892fa8a600e8edb6e25afac50852baf213b   
  Vulnerabilities │    0C     0H     2M    23L                                                 
  Pushed          │ 1 week ago                                                                 
  Size            │ 80 MB                                                                      
  Packages        │ 326                                                                        
  Runtime         │ 22                                                                         


Refresh base image
  Rebuild the image using a newer base image version. Updating this may result in breaking changes.

  ✓ This image version is up to date.


Change base image
  The list displays new recommended tags in descending order, where the top results are rated as most suitable.


                    Tag                    │                  Details                   │   Pushed   │       Vulnerabilities        
───────────────────────────────────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────
   22-slim                                 │ Benefits:                                  │ 1 week ago │    0C     0H     2M    23L   
  Image has same number of vulnerabilities │ • Same OS detected                         │            │                              
  Also known as:                           │ • Image is smaller by 1.7 MB               │            │                              
  • 22.14.0-slim                           │ • Image has same number of vulnerabilities │            │                              
  • 22.14-slim                             │ • Image contains equal number of packages  │            │                              
  • lts-slim                               │ • Tag is using slim variant                │            │                              
  • jod-slim                               │                                            │            │                              
  • 22-bookworm-slim                       │ Image details:                             │            │                              
  • jod-bookworm-slim                      │ • Size: 78 MB                              │            │                              
  • lts-bookworm-slim                      │ • Runtime: 22.14.0                         │            │                              
  • 22.14-bookworm-slim                    │                                            │            │                              
  • 22.14.0-bookworm-slim                  │                                            │            │                              
                                           │                                            │            │                              
                                           │                                            │            │                              
```
{{< /details >}}

# Сравнение с Trivy

Остановимся. Выдохнем. Подведем подитог. Я обновил базовый образ с `node:22` на `node:23-slim`, уменьшив его на 302 МБ и избавившись от 114 угроз. Docker Scout на текущем эапе находит 3 Medium и 23 Low угрозы. Сканируя тот же образ с помощью Trivy можно обнаружить 1 Critical, 1 High, 19 Medium и 57 Low угроз. Trivy находит значительно больше угроз.

{{< details summary="Обнаруженные Docker Scout угрозы в образе на основе `23-slim`" >}}
```shell
❯ docker scout cves local://vitkhab/serverless:3.40.0-23-slim
    ✓ SBOM of image already cached, 912 packages indexed
    ✗ Detected 14 vulnerable packages with a total of 26 vulnerabilities


## Overview

                    │           Analyzed Image             
────────────────────┼──────────────────────────────────────
  Target            │  local://vitkhab/serverless:3.40.0-23-slim 
    digest          │  c38eb2f9940a                        
    platform        │ linux/arm64/v8                       
    vulnerabilities │    0C     0H     3M    23L           
    size            │ 186 MB                               
    packages        │ 912                                  
                    │                                      
  Base image        │  node:23-slim                        
                    │  2dbdc1134da5                        


## Packages and Vulnerabilities

   0C     0H     1M     1L  libgnutls30 3.7.9-2+deb12u3
pkg:deb/debian/libgnutls30@3.7.9-2%2Bdeb12u3?arch=arm64&distro=debian-12&upstream=gnutls28

Dockerfile (1:1)
FROM node:23-slim

    ✗ MEDIUM CVE-2024-12243
      https://scout.docker.com/v/CVE-2024-12243
      Affected range : <3.7.9-2+deb12u4  
      Fixed version  : 3.7.9-2+deb12u4   
    
    ✗ LOW CVE-2011-3389
      https://scout.docker.com/v/CVE-2011-3389
      Affected range : >=3.7.9-2+deb12u3  
      Fixed version  : not fixed          
    

   0C     0H     1M     0L  axios 0.28.1
pkg:npm/axios@0.28.1

Dockerfile (14:14)
RUN npm install -g serverless@${SERVELESS_VERSION} "@yandex-cloud/serverless-plugin@${YC_PLUGIN_VERSION}"

    ✗ MEDIUM CVE-2023-45857 [OWASP Top Ten 2017 Category A9 - Using Components with Known Vulnerabilities]
      https://scout.docker.com/v/CVE-2023-45857
      Affected range : >=0.8.1                                       
                     : <1.6.0                                        
      Fixed version  : 1.6.0                                         
      CVSS Score     : 6.5                                           
      CVSS Vector    : CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N  
    

   0C     0H     1M     0L  libtasn1-6 4.19.0-2
pkg:deb/debian/libtasn1-6@4.19.0-2?arch=arm64&distro=debian-12

Dockerfile (1:1)
FROM node:23-slim

    ✗ MEDIUM CVE-2024-12133
      https://scout.docker.com/v/CVE-2024-12133
      Affected range : <4.19.0-2+deb12u1  
      Fixed version  : 4.19.0-2+deb12u1   
    

   0C     0H     0M     7L  libc6 2.36-9+deb12u9
pkg:deb/debian/libc6@2.36-9%2Bdeb12u9?arch=arm64&distro=debian-12&upstream=glibc

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2019-9192
      https://scout.docker.com/v/CVE-2019-9192
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2019-1010025
      https://scout.docker.com/v/CVE-2019-1010025
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2019-1010024
      https://scout.docker.com/v/CVE-2019-1010024
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2019-1010023
      https://scout.docker.com/v/CVE-2019-1010023
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2019-1010022
      https://scout.docker.com/v/CVE-2019-1010022
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2018-20796
      https://scout.docker.com/v/CVE-2018-20796
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    
    ✗ LOW CVE-2010-4756
      https://scout.docker.com/v/CVE-2010-4756
      Affected range : >=2.36-9+deb12u9  
      Fixed version  : not fixed         
    

   0C     0H     0M     4L  libudev1 252.33-1~deb12u1
pkg:deb/debian/libudev1@252.33-1~deb12u1?arch=arm64&distro=debian-12&upstream=systemd

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2023-31439
      https://scout.docker.com/v/CVE-2023-31439
      Affected range : >=252.33-1~deb12u1  
      Fixed version  : not fixed           
    
    ✗ LOW CVE-2023-31438
      https://scout.docker.com/v/CVE-2023-31438
      Affected range : >=252.33-1~deb12u1  
      Fixed version  : not fixed           
    
    ✗ LOW CVE-2023-31437
      https://scout.docker.com/v/CVE-2023-31437
      Affected range : >=252.33-1~deb12u1  
      Fixed version  : not fixed           
    
    ✗ LOW CVE-2013-4392
      https://scout.docker.com/v/CVE-2013-4392
      Affected range : >=252.33-1~deb12u1  
      Fixed version  : not fixed           
    

   0C     0H     0M     2L  libstdc++6 12.2.0-14
pkg:deb/debian/libstdc%2B%2B6@12.2.0-14?arch=arm64&distro=debian-12&upstream=gcc-12

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2023-4039
      https://scout.docker.com/v/CVE-2023-4039
      Affected range : >=12.2.0-14  
      Fixed version  : not fixed    
    
    ✗ LOW CVE-2022-27943
      https://scout.docker.com/v/CVE-2022-27943
      Affected range : >=12.2.0-14  
      Fixed version  : not fixed    
    

   0C     0H     0M     2L  perl-base 5.36.0-7+deb12u1
pkg:deb/debian/perl-base@5.36.0-7%2Bdeb12u1?arch=arm64&distro=debian-12&upstream=perl

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2023-31486
      https://scout.docker.com/v/CVE-2023-31486
      Affected range : >=5.36.0-7+deb12u1  
      Fixed version  : not fixed           
    
    ✗ LOW CVE-2011-4116
      https://scout.docker.com/v/CVE-2011-4116
      Affected range : >=5.36.0-7+deb12u1  
      Fixed version  : not fixed           
    

   0C     0H     0M     1L  coreutils 9.1-1
pkg:deb/debian/coreutils@9.1-1?arch=arm64&distro=debian-12

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2017-18018
      https://scout.docker.com/v/CVE-2017-18018
      Affected range : >=9.1-1    
      Fixed version  : not fixed  
    

   0C     0H     0M     1L  tar 1.34+dfsg-1.2+deb12u1
pkg:deb/debian/tar@1.34%2Bdfsg-1.2%2Bdeb12u1?arch=arm64&distro=debian-12

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2005-2541
      https://scout.docker.com/v/CVE-2005-2541
      Affected range : >=1.34+dfsg-1.2+deb12u1  
      Fixed version  : not fixed                
    

   0C     0H     0M     1L  libgcrypt20 1.10.1-3
pkg:deb/debian/libgcrypt20@1.10.1-3?arch=arm64&distro=debian-12

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2018-6829
      https://scout.docker.com/v/CVE-2018-6829
      Affected range : >=1.10.1-3  
      Fixed version  : not fixed   
    

   0C     0H     0M     1L  libapt-pkg6.0 2.6.1
pkg:deb/debian/libapt-pkg6.0@2.6.1?arch=arm64&distro=debian-12&upstream=apt

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2011-3374
      https://scout.docker.com/v/CVE-2011-3374
      Affected range : >=2.6.1    
      Fixed version  : not fixed  
    

   0C     0H     0M     1L  passwd 1:4.13+dfsg1-1+b1
pkg:deb/debian/passwd@1:4.13%2Bdfsg1-1%2Bb1?arch=arm64&distro=debian-12&upstream=shadow%401:4.13%2Bdfsg1-1

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2007-5686
      https://scout.docker.com/v/CVE-2007-5686
      Affected range : >=1:4.13+dfsg1-1  
      Fixed version  : not fixed         
    

   0C     0H     0M     1L  util-linux-extra 2.38.1-5+deb12u3
pkg:deb/debian/util-linux-extra@2.38.1-5%2Bdeb12u3?arch=arm64&distro=debian-12&upstream=util-linux

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2022-0563
      https://scout.docker.com/v/CVE-2022-0563
      Affected range : >=2.38.1-5+deb12u3  
      Fixed version  : not fixed           
    

   0C     0H     0M     1L  gpgv 2.2.40-1.1
pkg:deb/debian/gpgv@2.2.40-1.1?arch=arm64&distro=debian-12&upstream=gnupg2

Dockerfile (1:1)
FROM node:23-slim

    ✗ LOW CVE-2022-3219
      https://scout.docker.com/v/CVE-2022-3219
      Affected range : >=2.2.40-1.1  
      Fixed version  : not fixed     
    


26 vulnerabilities found in 14 packages
  CRITICAL  0   
  HIGH      0   
  MEDIUM    3   
  LOW       23 
```
{{< /details >}}


{{< details summary="Лог сканирования Trivy образа на основе `23-slim`" >}}
```shell
❯ docker run -ti -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image vitkhab/serverless:3.40.0-23-slim
2025-02-23T20:06:01Z    INFO    [vulndb] Need to update DB
2025-02-23T20:06:01Z    INFO    [vulndb] Downloading vulnerability DB...
2025-02-23T20:06:01Z    INFO    [vulndb] Downloading artifact...      repo="mirror.gcr.io/aquasec/trivy-db:2"
59.40 MiB / 59.40 MiB [-------------] 100.00% 5.25 MiB p/s 12s
2025-02-23T20:06:14Z    INFO    [vulndb] Artifact successfully downloaded     repo="mirror.gcr.io/aquasec/trivy-db:2"
2025-02-23T20:06:14Z    INFO    [vuln] Vulnerability scanning is enabled
2025-02-23T20:06:14Z    INFO    [secret] Secret scanning is enabled
2025-02-23T20:06:14Z    INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2025-02-23T20:06:14Z    INFO    [secret] Please see also https://aquasecurity.github.io/trivy/v0.59/docs/scanner/secret#recommendation for faster secret detection
2025-02-23T20:06:30Z    WARN    [secret] The size of the scanned file is too large. It is recommended to use `--skip-files` for this file to avoid high memory consumption. file_path="root/.npm/_cacache/content-v2/sha512/fb/d4/a23a8bfe2fc03c37fc207d930a2743aafb8c6ebecc3b29b6aa40020751f95d3f6491cc32fb6e30e9ff503cc8aba0be9f43200465ceccd78a640fc56fa501" size (MB)=16
2025-02-23T20:06:42Z    INFO    [python] Licenses acquired from one or more METADATA files may be subject to additional terms. Use `--debug` flag to see all affected packages.
2025-02-23T20:06:42Z    INFO    Detected OS     family="debian" version="12.9"
2025-02-23T20:06:42Z    INFO    [debian] Detecting vulnerabilities... os_version="12" pkg_num=88
2025-02-23T20:06:42Z    INFO    Number of language-specific files     num=2
2025-02-23T20:06:42Z    INFO    [python-pkg] Detecting vulnerabilities...
2025-02-23T20:06:42Z    INFO    [node-pkg] Detecting vulnerabilities...
2025-02-23T20:06:42Z    WARN    Using severities from other vendors for some vulnerabilities. Read https://aquasecurity.github.io/trivy/v0.59/docs/scanner/vulnerability#severity-selection for details.

vitkhab/serverless:3.40.0-23-slim (debian 12.9)

Total: 78 (UNKNOWN: 0, LOW: 57, MEDIUM: 19, HIGH: 1, CRITICAL: 1)

┌────────────────────┬─────────────────────┬──────────┬──────────────┬───────────────────────┬──────────────────┬──────────────────────────────────────────────────────────────┐
│      Library       │    Vulnerability    │ Severity │    Status    │   Installed Version   │  Fixed Version   │                            Title                             │
├────────────────────┼─────────────────────┼──────────┼──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ apt                │ CVE-2011-3374       │ LOW      │ affected     │ 2.6.1                 │                  │ It was found that apt-key in apt, all versions, do not       │
│                    │                     │          │              │                       │                  │ correctly...                                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2011-3374                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ bash               │ TEMP-0841856-B18BAF │          │              │ 5.2.15-2+b7           │                  │ [Privilege escalation possible to other user than root]      │
│                    │                     │          │              │                       │                  │ https://security-tracker.debian.org/tracker/TEMP-0841856-B1- │
│                    │                     │          │              │                       │                  │ 8BAF                                                         │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ bsdutils           │ CVE-2022-0563       │          │              │ 1:2.38.1-5+deb12u3    │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┤          ├──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ coreutils          │ CVE-2016-2781       │          │ will_not_fix │ 9.1-1                 │                  │ coreutils: Non-privileged session can escape to the parent   │
│                    │                     │          │              │                       │                  │ session in chroot                                            │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2016-2781                    │
│                    ├─────────────────────┤          ├──────────────┤                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2017-18018      │          │ affected     │                       │                  │ coreutils: race condition vulnerability in chown and chgrp   │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2017-18018                   │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ gcc-12-base        │ CVE-2022-27943      │          │              │ 12.2.0-14             │                  │ binutils: libiberty/rust-demangle.c in GNU GCC 11.2 allows   │
│                    │                     │          │              │                       │                  │ stack exhaustion in demangle_const                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-27943                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-4039       │          │              │                       │                  │ gcc: -fstack-protector fails to guard dynamic stack          │
│                    │                     │          │              │                       │                  │ allocations on ARM64                                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-4039                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ gpgv               │ CVE-2022-3219       │          │              │ 2.2.40-1.1            │                  │ gnupg: denial of service issue (resource consumption) using  │
│                    │                     │          │              │                       │                  │ compressed packets                                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-3219                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libapt-pkg6.0      │ CVE-2011-3374       │          │              │ 2.6.1                 │                  │ It was found that apt-key in apt, all versions, do not       │
│                    │                     │          │              │                       │                  │ correctly...                                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2011-3374                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libblkid1          │ CVE-2022-0563       │          │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libc-bin           │ CVE-2025-0395       │ MEDIUM   │              │ 2.36-9+deb12u9        │                  │ glibc: buffer overflow in the GNU C Library's assert()       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2025-0395                    │
│                    ├─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2010-4756       │ LOW      │              │                       │                  │ glibc: glob implementation can cause excessive CPU and       │
│                    │                     │          │              │                       │                  │ memory consumption due to...                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2010-4756                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2018-20796      │          │              │                       │                  │ glibc: uncontrolled recursion in function                    │
│                    │                     │          │              │                       │                  │ check_dst_limits_calc_pos_1 in posix/regexec.c               │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2018-20796                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010022    │          │              │                       │                  │ glibc: stack guard protection bypass                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010022                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010023    │          │              │                       │                  │ glibc: running ldd on malicious ELF leads to code execution  │
│                    │                     │          │              │                       │                  │ because of...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010023                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010024    │          │              │                       │                  │ glibc: ASLR bypass using cache of thread stack and heap      │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010024                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010025    │          │              │                       │                  │ glibc: information disclosure of heap addresses of           │
│                    │                     │          │              │                       │                  │ pthread_created thread                                       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010025                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-9192       │          │              │                       │                  │ glibc: uncontrolled recursion in function                    │
│                    │                     │          │              │                       │                  │ check_dst_limits_calc_pos_1 in posix/regexec.c               │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-9192                    │
├────────────────────┼─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│ libc6              │ CVE-2025-0395       │ MEDIUM   │              │                       │                  │ glibc: buffer overflow in the GNU C Library's assert()       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2025-0395                    │
│                    ├─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2010-4756       │ LOW      │              │                       │                  │ glibc: glob implementation can cause excessive CPU and       │
│                    │                     │          │              │                       │                  │ memory consumption due to...                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2010-4756                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2018-20796      │          │              │                       │                  │ glibc: uncontrolled recursion in function                    │
│                    │                     │          │              │                       │                  │ check_dst_limits_calc_pos_1 in posix/regexec.c               │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2018-20796                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010022    │          │              │                       │                  │ glibc: stack guard protection bypass                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010022                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010023    │          │              │                       │                  │ glibc: running ldd on malicious ELF leads to code execution  │
│                    │                     │          │              │                       │                  │ because of...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010023                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010024    │          │              │                       │                  │ glibc: ASLR bypass using cache of thread stack and heap      │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010024                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-1010025    │          │              │                       │                  │ glibc: information disclosure of heap addresses of           │
│                    │                     │          │              │                       │                  │ pthread_created thread                                       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-1010025                 │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2019-9192       │          │              │                       │                  │ glibc: uncontrolled recursion in function                    │
│                    │                     │          │              │                       │                  │ check_dst_limits_calc_pos_1 in posix/regexec.c               │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2019-9192                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libcap2            │ CVE-2025-1390       │ MEDIUM   │              │ 1:2.66-4              │                  │ libcap: pam_cap: Fix potential configuration parsing error   │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2025-1390                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libgcc-s1          │ CVE-2022-27943      │ LOW      │              │ 12.2.0-14             │                  │ binutils: libiberty/rust-demangle.c in GNU GCC 11.2 allows   │
│                    │                     │          │              │                       │                  │ stack exhaustion in demangle_const                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-27943                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-4039       │          │              │                       │                  │ gcc: -fstack-protector fails to guard dynamic stack          │
│                    │                     │          │              │                       │                  │ allocations on ARM64                                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-4039                    │
├────────────────────┼─────────────────────┼──────────┼──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libgcrypt20        │ CVE-2024-2236       │ MEDIUM   │ fix_deferred │ 1.10.1-3              │                  │ libgcrypt: vulnerable to Marvin Attack                       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-2236                    │
│                    ├─────────────────────┼──────────┼──────────────┤                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2018-6829       │ LOW      │ affected     │                       │                  │ libgcrypt: ElGamal implementation doesn't have semantic      │
│                    │                     │          │              │                       │                  │ security due to incorrectly encoded plaintexts...            │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2018-6829                    │
├────────────────────┼─────────────────────┼──────────┼──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libgnutls30        │ CVE-2024-12243      │ MEDIUM   │ fixed        │ 3.7.9-2+deb12u3       │ 3.7.9-2+deb12u4  │ gnutls: GnuTLS Impacted by Inefficient DER Decoding in       │
│                    │                     │          │              │                       │                  │ libtasn1 Leading to Remote...                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-12243                   │
│                    ├─────────────────────┼──────────┼──────────────┤                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2011-3389       │ LOW      │ affected     │                       │                  │ HTTPS: block-wise chosen-plaintext attack against SSL/TLS    │
│                    │                     │          │              │                       │                  │ (BEAST)                                                      │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2011-3389                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libmount1          │ CVE-2022-0563       │          │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libpam-modules     │ CVE-2024-10041      │ MEDIUM   │              │ 1.5.2-6+deb12u1       │                  │ pam: libpam: Libpam vulnerable to read hashed password       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-10041                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-22365      │          │              │                       │                  │ pam: allowing unprivileged user to block another user        │
│                    │                     │          │              │                       │                  │ namespace                                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-22365                   │
├────────────────────┼─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│ libpam-modules-bin │ CVE-2024-10041      │          │              │                       │                  │ pam: libpam: Libpam vulnerable to read hashed password       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-10041                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-22365      │          │              │                       │                  │ pam: allowing unprivileged user to block another user        │
│                    │                     │          │              │                       │                  │ namespace                                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-22365                   │
├────────────────────┼─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│ libpam-runtime     │ CVE-2024-10041      │          │              │                       │                  │ pam: libpam: Libpam vulnerable to read hashed password       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-10041                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-22365      │          │              │                       │                  │ pam: allowing unprivileged user to block another user        │
│                    │                     │          │              │                       │                  │ namespace                                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-22365                   │
├────────────────────┼─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│ libpam0g           │ CVE-2024-10041      │          │              │                       │                  │ pam: libpam: Libpam vulnerable to read hashed password       │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-10041                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-22365      │          │              │                       │                  │ pam: allowing unprivileged user to block another user        │
│                    │                     │          │              │                       │                  │ namespace                                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-22365                   │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libsmartcols1      │ CVE-2022-0563       │ LOW      │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libstdc++6         │ CVE-2022-27943      │          │              │ 12.2.0-14             │                  │ binutils: libiberty/rust-demangle.c in GNU GCC 11.2 allows   │
│                    │                     │          │              │                       │                  │ stack exhaustion in demangle_const                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-27943                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-4039       │          │              │                       │                  │ gcc: -fstack-protector fails to guard dynamic stack          │
│                    │                     │          │              │                       │                  │ allocations on ARM64                                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-4039                    │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libsystemd0        │ CVE-2013-4392       │          │              │ 252.33-1~deb12u1      │                  │ systemd: TOCTOU race condition when updating file            │
│                    │                     │          │              │                       │                  │ permissions and SELinux security contexts...                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2013-4392                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31437      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ modify a...                                                  │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31437                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31438      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ truncate a...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31438                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31439      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ modify the...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31439                   │
├────────────────────┼─────────────────────┼──────────┼──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libtasn1-6         │ CVE-2024-12133      │ MEDIUM   │ fixed        │ 4.19.0-2              │ 4.19.0-2+deb12u1 │ libtasn1: Inefficient DER Decoding in libtasn1 Leading to    │
│                    │                     │          │              │                       │                  │ Potential Remote DoS                                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-12133                   │
├────────────────────┼─────────────────────┤          ├──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libtinfo6          │ CVE-2023-50495      │          │ affected     │ 6.4-4                 │                  │ ncurses: segmentation fault via _nc_wrap_entry()             │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-50495                   │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libudev1           │ CVE-2013-4392       │ LOW      │              │ 252.33-1~deb12u1      │                  │ systemd: TOCTOU race condition when updating file            │
│                    │                     │          │              │                       │                  │ permissions and SELinux security contexts...                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2013-4392                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31437      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ modify a...                                                  │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31437                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31438      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ truncate a...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31438                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31439      │          │              │                       │                  │ An issue was discovered in systemd 253. An attacker can      │
│                    │                     │          │              │                       │                  │ modify the...                                                │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31439                   │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ libuuid1           │ CVE-2022-0563       │          │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ login              │ CVE-2023-4641       │ MEDIUM   │              │ 1:4.13+dfsg1-1+b1     │                  │ shadow-utils: possible password leak during passwd(1) change │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-4641                    │
│                    ├─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2007-5686       │ LOW      │              │                       │                  │ initscripts in rPath Linux 1 sets insecure permissions for   │
│                    │                     │          │              │                       │                  │ the /var/lo ......                                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2007-5686                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-29383      │          │              │                       │                  │ shadow: Improper input validation in shadow-utils package    │
│                    │                     │          │              │                       │                  │ utility chfn                                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-29383                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-56433      │          │              │                       │                  │ shadow-utils: Default subordinate ID configuration in        │
│                    │                     │          │              │                       │                  │ /etc/login.defs could lead to compromise                     │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-56433                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ TEMP-0628843-DBAD28 │          │              │                       │                  │ [more related to CVE-2005-4890]                              │
│                    │                     │          │              │                       │                  │ https://security-tracker.debian.org/tracker/TEMP-0628843-DB- │
│                    │                     │          │              │                       │                  │ AD28                                                         │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ mount              │ CVE-2022-0563       │          │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ ncurses-base       │ CVE-2023-50495      │ MEDIUM   │              │ 6.4-4                 │                  │ ncurses: segmentation fault via _nc_wrap_entry()             │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-50495                   │
├────────────────────┤                     │          │              │                       ├──────────────────┤                                                              │
│ ncurses-bin        │                     │          │              │                       │                  │                                                              │
│                    │                     │          │              │                       │                  │                                                              │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ passwd             │ CVE-2023-4641       │          │              │ 1:4.13+dfsg1-1+b1     │                  │ shadow-utils: possible password leak during passwd(1) change │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-4641                    │
│                    ├─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2007-5686       │ LOW      │              │                       │                  │ initscripts in rPath Linux 1 sets insecure permissions for   │
│                    │                     │          │              │                       │                  │ the /var/lo ......                                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2007-5686                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-29383      │          │              │                       │                  │ shadow: Improper input validation in shadow-utils package    │
│                    │                     │          │              │                       │                  │ utility chfn                                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-29383                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-56433      │          │              │                       │                  │ shadow-utils: Default subordinate ID configuration in        │
│                    │                     │          │              │                       │                  │ /etc/login.defs could lead to compromise                     │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2024-56433                   │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ TEMP-0628843-DBAD28 │          │              │                       │                  │ [more related to CVE-2005-4890]                              │
│                    │                     │          │              │                       │                  │ https://security-tracker.debian.org/tracker/TEMP-0628843-DB- │
│                    │                     │          │              │                       │                  │ AD28                                                         │
├────────────────────┼─────────────────────┼──────────┤              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ perl-base          │ CVE-2023-31484      │ HIGH     │              │ 5.36.0-7+deb12u1      │                  │ perl: CPAN.pm does not verify TLS certificates when          │
│                    │                     │          │              │                       │                  │ downloading distributions over HTTPS...                      │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31484                   │
│                    ├─────────────────────┼──────────┤              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2011-4116       │ LOW      │              │                       │                  │ perl: File:: Temp insecure temporary file handling           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2011-4116                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2023-31486      │          │              │                       │                  │ http-tiny: insecure TLS cert default                         │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-31486                   │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ sysvinit-utils     │ TEMP-0517018-A83CE6 │          │              │ 3.06-4                │                  │ [sysvinit: no-root option in expert installer exposes        │
│                    │                     │          │              │                       │                  │ locally exploitable security flaw]                           │
│                    │                     │          │              │                       │                  │ https://security-tracker.debian.org/tracker/TEMP-0517018-A8- │
│                    │                     │          │              │                       │                  │ 3CE6                                                         │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ tar                │ CVE-2005-2541       │          │              │ 1.34+dfsg-1.2+deb12u1 │                  │ tar: does not properly warn the user when extracting setuid  │
│                    │                     │          │              │                       │                  │ or setgid...                                                 │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2005-2541                    │
│                    ├─────────────────────┤          │              │                       ├──────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ TEMP-0290435-0B57B5 │          │              │                       │                  │ [tar's rmt command may have undesired side effects]          │
│                    │                     │          │              │                       │                  │ https://security-tracker.debian.org/tracker/TEMP-0290435-0B- │
│                    │                     │          │              │                       │                  │ 57B5                                                         │
├────────────────────┼─────────────────────┤          │              ├───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ util-linux         │ CVE-2022-0563       │          │              │ 2.38.1-5+deb12u3      │                  │ util-linux: partial disclosure of arbitrary files in chfn    │
│                    │                     │          │              │                       │                  │ and chsh when compiled...                                    │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2022-0563                    │
├────────────────────┤                     │          │              │                       ├──────────────────┤                                                              │
│ util-linux-extra   │                     │          │              │                       │                  │                                                              │
│                    │                     │          │              │                       │                  │                                                              │
│                    │                     │          │              │                       │                  │                                                              │
├────────────────────┼─────────────────────┼──────────┼──────────────┼───────────────────────┼──────────────────┼──────────────────────────────────────────────────────────────┤
│ zlib1g             │ CVE-2023-45853      │ CRITICAL │ will_not_fix │ 1:1.2.13.dfsg-1       │                  │ zlib: integer overflow and resultant heap-based buffer       │
│                    │                     │          │              │                       │                  │ overflow in zipOpenNewFileInZip4_6                           │
│                    │                     │          │              │                       │                  │ https://avd.aquasec.com/nvd/cve-2023-45853                   │
└────────────────────┴─────────────────────┴──────────┴──────────────┴───────────────────────┴──────────────────┴──────────────────────────────────────────────────────────────┘
```
{{< /details >}}

Полученный образ для локально запускаемой утилиты представляет из себя решето из 78 дыр информационной безопасности. Что меня не устраивает. Поэтому я продолжаю оптимизировать образ, перехожу на `node:23-alpine`. Изменение, которое Docker Scout не предложил. Trivy тоже. Итоговый `Dockerfile` выглядит так:
```dockerfile
FROM node:23-alpine

ARG SERVELESS_VERSION=3.40.0
ARG YC_PLUGIN_VERSION=1.7.25

# Install Serverless Framework globally and Yandex.Cloud plugin
RUN npm install -g \
    serverless@${SERVELESS_VERSION} \
    "@yandex-cloud/serverless-plugin@${YC_PLUGIN_VERSION}"{YC_PLUGIN_VERSION}"

WORKDIR /app

CMD [ "serverless" ]
```

Собираю. Проверяю с помощью Trivy. 0 найденных уязвимостей. Успех!

{{< details summary="Лог сканирования Trivy образа на основе `23-alpine`" >}}
```shell
❯ docker run -ti -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image vitkhab/serverless:3.40.0-alpine

2025-02-23T19:13:19Z    INFO    [vulndb] Need to update DB
2025-02-23T19:13:19Z    INFO    [vulndb] Downloading vulnerability DB...
2025-02-23T19:13:19Z    INFO    [vulndb] Downloading artifact...        repo="mirror.gcr.io/aquasec/trivy-db:2"
59.40 MiB / 59.40 MiB [--------------------------------------------------------------------------------------------------------------------] 100.00% 7.80 MiB p/s 7.8s
2025-02-23T19:13:28Z    INFO    [vulndb] Artifact successfully downloaded       repo="mirror.gcr.io/aquasec/trivy-db:2"
2025-02-23T19:13:28Z    INFO    [vuln] Vulnerability scanning is enabled
2025-02-23T19:13:28Z    INFO    [secret] Secret scanning is enabled
2025-02-23T19:13:28Z    INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2025-02-23T19:13:28Z    INFO    [secret] Please see also https://aquasecurity.github.io/trivy/v0.59/docs/scanner/secret#recommendation for faster secret detection
2025-02-23T19:13:40Z    WARN    [secret] The size of the scanned file is too large. It is recommended to use `--skip-files` for this file to avoid high memory consumption.     file_path="root/.npm/_cacache/content-v2/sha512/fb/d4/a23a8bfe2fc03c37fc207d930a2743aafb8c6ebecc3b29b6aa40020751f95d3f6491cc32fb6e30e9ff503cc8aba0be9f43200465ceccd78a640fc56fa501" size (MB)=16
2025-02-23T19:13:50Z    INFO    [python] Licenses acquired from one or more METADATA files may be subject to additional terms. Use `--debug` flag to see all affected packages.
2025-02-23T19:13:50Z    INFO    Detected OS     family="alpine" version="3.21.3"
2025-02-23T19:13:50Z    INFO    [alpine] Detecting vulnerabilities...   os_version="3.21" repository="3.21" pkg_num=17
2025-02-23T19:13:50Z    INFO    Number of language-specific files       num=2
2025-02-23T19:13:50Z    INFO    [python-pkg] Detecting vulnerabilities...
2025-02-23T19:13:50Z    INFO    [node-pkg] Detecting vulnerabilities...

vitkhab/serverless:3.40.0-alpine (alpine 3.21.3)

Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```
{{< /details >}}

Смотрю, что покажет Docker Scout. Найдена уязвимость в пакетах Node.js, которой нет в логе Trivy. А получилось так из-за ошибки парсинга [базы CVE Gitlab][7] в [базу Docker Scout][8]. В источнике указаны интервалы >= 1.0.0, < 1.6.0 и >= 0.8.1, < 0.28.0. Когда в целевой базе потерялась пара условий и итоговый интервал стал >=0.8.1, <1.6.0.

[7]: https://advisories.gitlab.com/pkg/npm/axios/CVE-2023-45857/
[8]: https://scout.docker.com/vulnerabilities/id/CVE-2023-45857?s=gitlab&n=axios&t=npm&vr=%3E%3D0.8.1%2C%3C1.6.0

А еще, одна из рекомендаций Docker Scout заменить базовый образ на `23-slim`, добавив 25 уязвимостей.

{{< details summary="Лог сканирования Docker Scout образа на основе `23-alpine`" >}}
```shell
❯ docker scout quickview local://vitkhab/serverless:3.40.0-alpine
    ✓ Image stored for indexing
    ✓ SBOM obtained from attestation, 841 packages found
    ✓ Provenance obtained from attestation
    ✓ Pulled

  Target             │  local://vitkhab/serverless:3.40.0-alpine  │    0C     0H     1M     0L   
    digest           │  6bb47c73e8d3                              │                              
  Base image         │  node:23-alpine                            │    0C     0H     0M     0L   
  Updated base image │  node:22-alpine                            │    0C     0H     0M     0L   
                     │                                            │                              

❯ docker scout cves local://vitkhab/serverless:3.40.0-alpine
    ✓ SBOM of image already cached, 841 packages indexed
    ✗ Detected 1 vulnerable package with 1 vulnerability


## Overview

                    │           Analyzed Image             
────────────────────┼──────────────────────────────────────
  Target            │  local://vitkhab/serverless:3.40.0-alpine
    digest          │  6bb47c73e8d3                        
    platform        │ linux/arm64/v8                       
    vulnerabilities │    0C     0H     1M     0L           
    size            │ 163 MB                               
    packages        │ 841                                  
                    │                                      
  Base image        │  node:23-alpine                      
                    │  3bc3c0c99ea8                        


## Packages and Vulnerabilities

   0C     0H     1M     0L  axios 0.28.1
pkg:npm/axios@0.28.1

Dockerfile (14:14)
RUN npm install -g serverless@${SERVELESS_VERSION} "@yandex-cloud/serverless-plugin@${YC_PLUGIN_VERSION}"

    ✗ MEDIUM CVE-2023-45857 [OWASP Top Ten 2017 Category A9 - Using Components with Known Vulnerabilities]
      https://scout.docker.com/v/CVE-2023-45857
      Affected range : >=0.8.1                                       
                     : <1.6.0                                        
      Fixed version  : 1.6.0                                         
      CVSS Score     : 6.5                                           
      CVSS Vector    : CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N  
    


1 vulnerability found in 1 package
  CRITICAL  0  
  HIGH      0  
  MEDIUM    1  
  LOW       0  

❯ docker scout recommendations local://vitkhab/serverless:3.40.0-alpine
    ✓ SBOM of image already cached, 841 packages indexed

  Target   │  local://vitkhab/serverless:3.40.0-alpine
    digest │  6bb47c73e8d3                        

## Recommended fixes

  Base image is  node:23-alpine 

  Name            │  23-alpine                                                                 
  Digest          │  sha256:3bc3c0c99ea8c3f62e11a88bd049981d69d2dc5d0752d1bd80c7b341aee6a8e4   
  Vulnerabilities │    0C     0H     0M     0L                                                 
  Pushed          │ 1 week ago                                                                 
  Size            │ 57 MB                                                                      
  Packages        │ 223                                                                        
  Flavor          │ alpine                                                                     
  OS              │ 3.21                                                                       
  Runtime         │ 22                                                                         


Refresh base image
  Rebuild the image using a newer base image version. Updating this may result in breaking changes.

  ✓ This image version is up to date.


Change base image
  The list displays new recommended tags in descending order, where the top results are rated as most suitable.


                    Tag                    │                  Details                   │   Pushed   │       Vulnerabilities        
───────────────────────────────────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────
   22-alpine                               │ Benefits:                                  │ 1 week ago │    0C     0H     0M     0L   
  Image has same number of vulnerabilities │ • Same OS detected                         │            │                              
  Also known as:                           │ • Image is smaller by 1.7 MB               │            │                              
  • 22.14.0-alpine                         │ • Image has same number of vulnerabilities │            │                              
  • 22.14.0-alpine3.21                     │ • Image contains equal number of packages  │            │                              
  • 22.14-alpine                           │                                            │            │                              
  • 22.14-alpine3.21                       │ Image details:                             │            │                              
  • lts-alpine                             │ • Size: 55 MB                              │            │                              
  • lts-alpine3.21                         │ • Flavor: alpine                           │            │                              
  • 22-alpine3.21                          │ • OS: 3.21                                 │            │                              
  • jod-alpine                             │ • Runtime: 22.14.0                         │            │                              
  • jod-alpine3.21                         │                                            │            │                              
                                           │                                            │            │                              
                                           │                                            │            │                              
   slim                                    │ Benefits:                                  │ 1 week ago │    0C     0H     2M    23L   
  Tag is preferred tag                     │ • Tag is preferred tag                     │            │                  +2    +23   
  Also known as:                           │ • Tag is using slim variant                │            │                              
  • 23.8.0-slim                            │ • slim was pulled 17K times last month     │            │                              
  • 23.8-slim                              │                                            │            │                              
  • current-slim                           │ Image details:                             │            │                              
  • 23-slim                                │ • Size: 80 MB                              │            │                              
  • bookworm-slim                          │ • Runtime: 22                              │            │                              
  • 23-bookworm-slim                       │                                            │            │                              
  • 23.8-bookworm-slim                     │                                            │            │                              
  • 23.8.0-bookworm-slim                   │                                            │            │                              
  • current-bookworm-slim                  │                                            │            │                              
                                           │                                            │            │                              
                                           │                                            │            │                              
```
{{< /details >}}

# Вывод

Я рад, что Docker из коробки предлагает мне просканировать образ на уязвимости. И даже подсказывает, что можно улучшить в силу своих возможностей. Это сильно лучше, чем ничего. Но в сравнении с Trivy он проигрывает по качеству сканирования и возможностям. Docker Scout не может заменить Trivy в промышленной разработке, но для небольших личных проектов сойдет.
