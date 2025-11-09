# Programowanie Full-Stack w Chmurze Obliczeniowej

      Marek Prokopiuk
      grupa dziekańska: 2.3
      numer albumu: 097710
## Laboratorium 5<br>Zarządzanie zasobami. Deklarowanie żądań i limitów zasobów. Wykorzystanie obiektów Quota, LimitRange oraz Deployment. 
<p align="justify">Przedstawione zostało rozwiązanie zadania do laboratorium 5 dotyczącego zarządzania zasobami w klastrze Kubernetes. Celem ćwiczenia było stworzenie i uruchomienie plików konfiguracyjnych definiujących podział i ograniczenia zasobów zgodnie z przyjętymi założeniami, a następnie ich przetestowanie przy użyciu obiektów typu <i>Deployment</i>. </p>

---

### Utworzenie przestrzeni nazw i ustalenie ich zasobów
<p align="justify"> Na początku uruchomiony został klaster Kubernetes w środowisku Minikube. Utworzono dwie dedykowane przestrzenie nazw: <code>ns-dev</code> dla środowiska rozwojowego oraz <code>ns-prod</code> jako środowisko produkcyjne. Każdej przestrzeni przypisano odpowiednie limity zasobów przy użyciu obiektów <i>Quota</i>. Dla <code>ns-dev</code> ustalono, że łączna liczba uruchomionych Pod-ów nie może przekroczyć 10, a całkowite wartości zadeklarowanych żądań (<i>requests</i>) i limitów (<i>limits</i>) CPU oraz pamięci są odpowiednio ograniczone. Łączna suma <i>requests.cpu</i> dla wszystkich Pod-ów w przestrzeni nazw nie może przekroczyć 500m (0.5 CPU), a <i>requests.memory</i> — 500Mi. Suma <i>limits.cpu</i> została ograniczona natomiast do 1000m (1 CPU), a <i>limits.memory</i> do 1000Mi. W <code>ns-prod</code> wartości te zostały ustawione na poziomie dwukrotnie wyższym, zgodnie z wymaganiami zadania. Oznacza to, że Kubernetes nie pozwoli uruchomić nowych Pod-ów, jeśli ich zadeklarowane wartości zasobów spowodowałyby przekroczenie tych limitów w skali całego namespace’u. Warto podkreślić, że obiekt <i>Quota</i> nie kontroluje zasobów pojedynczych kontenerów, lecz sumaryczne zużycie w ramach przestrzeni nazw. Na poniższych zrzutach ekrany zostało pokazane utworzenie przestrzeni nazw i odpowiednich obiektów <i>Quota</i>.</p> 
<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/1.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 1. Utworzenie przestrzeni nazw</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/2.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 2. Utworzenie obiektów Quota dla każdej przestrzeni nazw</i>
</p>

<p align="justify"> Na tym etapie <i>Quota</i> wymaga, aby każdy uruchamiany Pod miał jawnie zadeklarowane żądania i limity zasobów (requests i limits). Aby umożliwić uruchamianie aplikacji w namespace <code>ns-dev</code> bez definiowania żądań i limitów zasobów, utworzony został obiekt <i>LimitRange</i>. Plik konfiguracyjny <code>ns-dev-limitrange.yaml</code> definiuje domyślne wartości oraz maksymalne dopuszczalne limity dla pojedynczych kontenerów. Domyślne żądania (<i>defaultRequest</i>) ustawiono na 100m CPU i 128Mi pamięci, natomiast maksymalne wartości (<i>max</i>) ograniczono do 200m CPU i 256Mi pamięci (zgodnie z parametrami podanymi w poleceniu). Dzięki temu każdy Pod uruchamiany w <code>ns-dev</code>, nawet jeśli w manifeście nie zadeklaruje swoich zasobów, otrzyma wartości domyślne, nie przekraczając ustalonych limitów. Obiekt <i>LimitRange</i> działa na poziomie pojedynczych kontenerów w namespace i nie zastępuje obiektu <i>Quota</i>, który nadal kontroluje sumaryczne zużycie CPU, pamięci oraz maksymalną liczbę uruchomionych Pod-ów (tutaj ustawioną na 10). Po dodaniu <i>LimitRange</i> każdy Pod w <code>ns-dev</code>:</p> <ul> <li>może być uruchamiany bez deklarowania własnych zasobów,</li> <li>otrzymuje domyślne żądania CPU i pamięci (100m / 128Mi),</li> <li>nie przekracza maksymalnych limitów (200m CPU / 256Mi RAM).</li> </ul> <p align="justify">W ten sposób obiekty <i>Quota</i> i <i>LimitRange</i> współdziałają, zapewniając pełną kontrolę nad zużyciem zasobów w namespace <code>ns-dev</code> — <i>Quota</i> ogranicza sumaryczne wykorzystanie CPU, pamięci i liczbę Pod-ów, natomiast <i>LimitRange</i> narzuca domyślne i maksymalne wartości dla pojedynczych kontenerów. Na poniższych zrzutach ekranu pokazany został omówiony plik konfiguracyjny, a także wykonane polecenia ustawiające <i>LimitRange</i> dla odpowiedniej przestrzeni nazw.</p>
<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/3.png" style="width: 50%; height: 50%" /></p>
<p align="center">
  <i>Rys. 3. Zawartość pliku ns-dev-limitrange.yaml</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/4.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 4. Polecenia tworzące LimitRange w przestrzeni nazw</i>
</p>

---

### Testowanie z wykorzystaniem obiektów Deployment
<p align="justify"> W celu przetestowania, czy zarządzanie zasobami w przestrzeni <code>ns-dev</code> zostało poprawnie zrealizowane, utworzone zostały trzy różne obiekty <i>Deployment</i>. Najpierw został utworzony i uruchomiony obiekt o nazwie <i>no-test</i> na bazie obrazu nginx, który przekroczył określone wymagania, co do wykorzystania zasobów. Ustawione zostało, że <i>Deployment</i> zawierał będzie 5 Pod-ów (replicas: 5), a także miał ustawione żądania i limity CPU i pamięci odpowiednio na 300m i 512Mi (co rzecz jasna przekracza wartości z obiektu <i>LimitRange</i>, czyli 200m i 256Mi). Po uruchomieniu <i>Deploymentu</i> okazało się, że on istnieje, ale żadne Pod-y nie są uruchomione (0/5 dostępnych). Dokładnie taki efekt chcieliśmy dla testu <i>no-test</i >— <i>Deployment</i> próbuje przekroczyć Limity <i>LimitRange</i>, a Kubernetes blokuje tworzenie Pod-ów. Pod-y nie startują, bo celowo przekraczają zasoby. Aby udowodnić, że z tego powodu <i>no-test</i> nie działa poprawnie, użyte zostały między innymi polecenia:</p> 

```diff
kubectl get all -l app=no-test -n ns-dev
kubectl describe rs no-test-f7d5bfd44 -n ns-dev
```

<p align="justify"> Pozwoliły one udowodnić, że przyczyną nieprawidłowego działania uruchomionego obiektu jest przekroczenie limitów ustawionych w <i>LimitRange</i>. Widać to w wyświetlonych zdarzeniach, co zostało pokazane na poniższych zrzutach ekranu.</p> 

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/5_no-test.png" style="width: 65%; height: 65%" /></p>
<p align="center">
  <i>Rys. 5. Zawartość pliku no-test.yaml z ustawionymi żądaniami i limitami zasobów</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/6_no-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 6. Utworzenie obiektu Deployment no-test i pokazanie, że nie działa prawidłowo</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/7_no-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 7. Events, udowadniające z jakiego powodu obiekt nie działa poprawnie </i>
</p>

<p align="justify"> Następnie został utworzony i uruchomiony kolejny obiekt <i>Deployment</i> o nazwie <i>yes-test</i>, również na bazie obrazu nginx i w przestrzeni nazw <code>ns-dev</code>. Tym razem jednak obiekt ten spełniał wymagania dotyczące wykorzystania zasobów. Ustawione zostało, że będzie zawierał 3 Pod-y (replicas: 3), a także będzie miał ustawione żądania CPU i pamięci odpowiednio na 100m i 128Mi, natomiast limity CPU i pamięci na 200m i 256Mi. Obiekt <i>yes-test</i> został utworzony poprawnie, ponieważ jego zasoby mieszczą się w limitach <i>LimitRange</i>. Wszystkie Pod-y uruchomiły się i są w stanie Running, co potwierdza poprawną konfigurację zasobów.</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/8_yes-test.png" style="width: 65%; height: 65%" /></p>
<p align="center">
  <i>Rys. 8. Zawartość pliku yes-test.yaml z ustawionymi żądaniami i limitami zasobów</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/9_yes-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 9. Utworzenie obiektu Deployment o nazwie yes-test i pokazanie, że działa prawidłowo</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/10_yes-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 10. Events, udowadniające prawidłowe działanie obiektu</i>
</p>

<p align="justify"> Ostatni test polegał na utworzeniu i uruchomieniu <i>Deploymentu</i> o nazwie <i>zero-test</i>, takiego samego jak w poprzednich testach, jednak z tą różnicą, że miał on nie posiadać deklaracji odnośnie żądań i limitów zasobów. Początkowo jednak usunięty został obiekt <i>yes-test</i>, aby umożliwić uruchomienie obiektu <i>zero-test</i>, ponieważ łączna wartość zadeklarowanych zasobów w przestrzeni nazw <code>ns-dev</code> przekraczała limity określone w <i>Quota</i>. Zbyt duże zapotrzebowanie na pamięć uniemożliwiało utworzenie nowych Pod-ów, dlatego konieczne było albo zmodyfikowanie quoty, albo — co zostało zrobione — zwolnienie części zasobów poprzez usunięcie istniejącego <i>Deploymentu</i>. Po wykonaniu tych czynności obiekt <i>zero-test</i> uruchomił się poprawnie, a wszystkie Pod-y miały status Running. Analiza szczegółów Pod-ów wykazała, że <i>LimitRanger</i> automatycznie przypisał im wartości CPU i pamięci zgodne z domyślnymi ustawieniami określonymi w obiekcie <i>LimitRange</i> (requests: 100m / 128Mi, limits: 100m / 128Mi). Potwierdza to, że <i>LimitRange</i>  działa poprawnie i zapewnia przydział domyślnych zasobów w przypadku ich braku w specyfikacji kontenera. Na poniższych zrzutach ekran został pokazany wykorzystany w tym teście plik konfiguracyjny oraz wykorzystane polecenia.</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/11_zero-test.png" style="width: 65%; height: 65%" /></p>
<p align="center">
  <i>Rys. 11. Zawartość pliku zero-test.yaml bez deklaracji odnośnie żądań i limitów zasobów</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/12_zero-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 12. Utworzenie obiektu Deployment o nazwie zero-test i pokazanie, że działa on poprawnie</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/13_zero-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 13. Events, udowadniające prawidłowe działanie obiektu</i>
</p>

<p align="center">
  <img src="https://github.com/MarekP21/PFSwChO_Lab5/blob/main/screeny/14_zero-test.png" style="width: 80%; height: 80%" /></p>
<p align="center">
  <i>Rys. 14. Szczegóły jednego z Pod-ów, w których widać domyślnie przypisane wartości zasobów</i>
</p>

---
