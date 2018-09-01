# Wstęp
Jeśli jeszcze nie wiesz co to jest php-fpm, jak działa to zapraszam do zapoznania się z 
innymi ciekawymi artykułami na ten temat, do których link znajdziesz na koncu 

Tematem tego artykułu jest poprawa wydajności i eleminacja problemów z którymi naktnąłem się podczas 
wdrażania rozwiązania opartego o PHP-FPM i Nginx. Jeśli przeanalizujesz swój serwer zgodnie z tym jak
zrobiłem to ja to będziesz miał gwarancję, że lepiej zrozumiesz jak działają rozwiązania które używasz
i będziesz miał nad nimi lepszą kontrolę.

# Random stuff
Gdy zaczynamy przygodę z nową technologią nie znamy jej dobrze i nigdy nie ma tak kolorowo, że wszystko
będzie działało tak idealnie jakbyśmy tego chcieli. Gdy pojawiają się pierwsze problemy często nie wiemy
jak sobie z nimi radzić i robimy **losowe rzeczy**, po czym sprawdzamy jaki to przyniosło rezultat. 

### Skutki?
Często takie podejście może wprowadzić więcej haosu i bałaganu. Mi osobiście w swojej pracy często zdarza
się stosować takie podejście, że wykonuje **random stuff** i czekam na cud, często zdarzyło mi się 
wygenerować w ten sposób dodatkowe problemy, przez co traciłem mase czasu, bo te dodatkowe problemy wymagały
później więcej czasu na określenie przyczyny i naprawę nowo powstałych problemów.

### Przyczyna
Jedną jedyną przyczyną tego jest to, że używamy to czego nie znamy - samo w sobie nie jest to złe podejście
ale wraz z rozwojem projektu/produktu należy zagłębiać się w określoną technologię aby ją zrozumieć.



# Zaczynamy
### Monitoring
Aby wiedzieć co w naszym systemie jest nie tak, musimy go monitorować. Monitoring jest jedną z najważniejszych 
rzeczy w naszym systemi bo to on mówi nam czy system działa czy nie i jakiej jest kondycji.

Sam proces monitoringu wymaga dodatkowego doświadczenia, które pozwoli określić kiedy i jak należy 
reagować.

# Konfiguracja Nginxa.

### Logi

##### Konfiguracja logów.

Dzisiaj standardem jest generowanie logów w formacie **JSON** pozwala to na łatwe ich filtrowanie i 
ewentualnie grupowanie, generowanie metryk na ich podstawie.

Najpierw definiujemy sobie format logów jaki będziemy używać. Aby to zrobić tworzymy plik
`/etc/nginx/conf.d/log_format.conf` o treści:

```conf
log_format get escape=json '{'
        '"time_ms":"$msec",'
        '"time_iso8601":"$time_iso8601",'
        '"ip":"$http_x_forwarded_for",'
        '"http_x_forwarded_proto":"$http_x_forwarded_proto",'
        '"status":"$status",'
        '"method":"$request_method",'
        '"scheme":"$scheme",'
        '"host":"$host",'
        '"path":"$request_uri",'
        '"query":"$query_string",'
        '"referrer_uri":"$http_referer",'
        '"user_agent":"$http_user_agent",'
        '"remote_user":"$remote_user",'
        '"remote_port":"$remote_port",'
        '"server_name":"$server_name",'
        '"server_port":"$server_port",'
        '"server_protocol":"$server_protocol",'
        '"nginx_version":"$nginx_version",'
        '"nginx_pid":"$pid",'
        '"nginx_time":"$request_time",'
        '"upstream_response_time":"$upstream_response_time",'
        '"upstream_connect_time":"$upstream_connect_time",'
        '"upstream_header_time":"$upstream_header_time",'
        '"connections_active":"$connections_active",'
        '"connection_reading":"$connections_reading",'
        '"connection_writings":"$connections_writing",'
        '"connections_waiting":"$connections_waiting"'
'}';
```
Wpis ten zawiera wszystkie informacje, które przydały się w mojej pracy z Nginxem. Po więcej informacji
zapraszam do [documentacji Nginxa](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format)



Czas aby aktywować nowy format logów dla naszej aplikacji. W tym celu otwieramy ustawienia nginxa dla naszej 
strony - prawdopodobnie będzie to jeden z plików znajdujących się w katalogu: `/etc/nginx/sites-enabled/`
lub w katalogu `/etc/nginx/conf.d`. Następnie włączamy logowanie w wcześniej zdefiniowanym formacie. 
Dodajemy w bloku server takie linie: 
```conf
error_log  /var/log/nginx/application.error.log warn;
access_log /var/log/nginx/application.access.log get;
``` 
Więcej informacji o powyższych dyrektywach w [dokumentacji](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) 


Sprawdzamy, czy nasza konfiguracja jest poprawna wpisując polecenie:
```
nginx -t
```
Jeśli wszystko jest poprawnie skonfigurowane powinniśmy zobaczyć poniższy komunikat.
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Teraz czas na załadowanie naszej nowej konfiguracji. W tym celu wykonujemy komendę:
```
nginx -s reload
```

I po chwili logi powinny zacząć pojawiać się w naszym formacie. Przykład logów z mojej aplikacji:
```
[daniel@archlinux tools]$ tail -n4 /var/log/nginx/application.access.log


{"time_ms":"1535473131.906","time_iso8601":"2018-08-28T19:18:51+03:00","ip":"-","http_x_forwarded_proto":"-","status":"200","method":"GET","scheme":"http","host":"local.application.com","path":"/themes/classic/images/fugue/stamp.png","query":"-","referrer_uri":"http://local.application.com:8282/overview.html","user_agent":"Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36","remote_user":"-","remote_port":"59720","server_name":"localhost","server_port":"80","server_protocol":"HTTP/1.1","nginx_version":"1.15.2","nginx_pid":"14","nginx_time":"0.000","upstream_response_time":"-","upstream_connect_time":"-","upstream_header_time":"-"}
{"time_ms":"1535473131.906","time_iso8601":"2018-08-28T19:18:51+03:00","ip":"-","http_x_forwarded_proto":"-","status":"200","method":"GET","scheme":"http","host":"local.application.com","path":"/themes/classic/images/scissors.png","query":"-","referrer_uri":"http://local.application.com:8282/overview.html","user_agent":"Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36","remote_user":"-","remote_port":"59724","server_name":"localhost","server_port":"80","server_protocol":"HTTP/1.1","nginx_version":"1.15.2","nginx_pid":"14","nginx_time":"0.000","upstream_response_time":"-","upstream_connect_time":"-","upstream_header_time":"-"}
{"time_ms":"1535473133.710","time_iso8601":"2018-08-28T19:18:53+03:00","ip":"-","http_x_forwarded_proto":"-","status":"302","method":"GET","scheme":"http","host":"local.application.com","path":"/logout.html","query":"","referrer_uri":"http://local.application.com:8282/overview.html","user_agent":"Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36","remote_user":"-","remote_port":"59724","server_name":"localhost","server_port":"80","server_protocol":"HTTP/1.1","nginx_version":"1.15.2","nginx_pid":"14","nginx_time":"0.109","upstream_response_time":"0.000, 0.110","upstream_connect_time":"0.000, 0.000","upstream_header_time":"0.000, 0.110"}
{"time_ms":"1535473133.860","time_iso8601":"2018-08-28T19:18:53+03:00","ip":"-","http_x_forwarded_proto":"-","status":"200","method":"GET","scheme":"http","host":"local.application.com","path":"/","query":"-","referrer_uri":"http://local.application.com:8282/overview.html","user_agent":"Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36","remote_user":"-","remote_port":"59720","server_name":"localhost","server_port":"80","server_protocol":"HTTP/1.1","nginx_version":"1.15.2","nginx_pid":"14","nginx_time":"0.146","upstream_response_time":"0.146","upstream_connect_time":"0.000","upstream_header_time":"0.146"}
```

##### Jak czytać logi.
Pierwszą rzeczą jest przeglądnięcie pliku /var/log/nginx/application.error.log. W nim będą wszystkie niepokojące informacje.
Ale ja robię to nieco inaczej. Takie szukanie informacji w error logu jest żmudne i często
to overwork...
Według mnie najlepszym sposobem jest szukanie w access logu jakichś niepoprawnych requestów.
W tym celu możemy użyć poniższej komendy:

```
cat /var/log/nginx/application.access.log | grep "time_ms" |  jq '. | select(.status >= "400")'
```

Wyświetli nam to wszystki requesty, które mają status większy bądź równy 400.
Jeśli znajdziemy już podejrzane wpisy -ajważniejsze są dla nas te o statusie 404, 500, 503, 502.
To możemy szukać korelacji z innymi elementami systemu. Np wtedy otwieramy log z błędami
i szukamy wiadomości o błędach w tym samym (+/- 2 sekundy) czasie w którym został dodany
wpis w access logu.



### Metryki

Skoro mamy skonfigurowane logi chcielibyśmy widzieć co się dzieje z naszym serwerem na jakims wykresie.
Najpierw spradzamy czy nasz Nginx jest skompilowany z modułem `stub_status`. W tym celu wykonujemy komendę

```
nginx -V 2>&1 | grep --color -- --with-http_stub_status_module
```

Jeśli widzimy pokolorowany tekst to znaczy, że mamy ten moduł. 

Teraz tworzymy nowy plik o nazwie `nginx-status.conf` w `/etc/nginx/sites-enabled/` lub w `/etc/nginx/conf.d`
Zawartość pliku:   
```
server {
    listen 127.0.0.1:80;
    server_name 127.0.0.1;

    location /nginx_status {
        stub_status;
    }
}
```

Przeładowujemy konfigurację:

```
[daniel@archlinux ~]$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[daniel@archlinux ~]$ sudo nginx -s reload
2018/09/01 17:11:13 [notice] 17836#17836: signal process started
```

Od teraz mamy dostęp do endpointu http://127.0.0.1/nginx_status, który zwraca nam informację:
```
curl http://127.0.0.1/nginx_status
Active connections: 2 
server accepts handled requests
 21211 21211 5321111 
Reading: 0 Writing: 1 Waiting: 1
```

##### Co te metryki oznaczają?

The following status information is provided:

* Active connections - Liczba aktywnych połączeń z Nginx'em wliczając w to połączenia oczekujące.
* accepts- Całkowita liczba zaakceptowanych requestów.
* handled - Całkowita liczba obsłużonych requestów. Liczba ta **powinna** być taka sama jak liczba zaakceptowanych połącze
* Reading - Liczba połączeń, w których Nginx aktualnie czyta nagłówki nadesłanych rządań.
* Writing - Liczba połączeń, w których Nginx wysyła odpowiedź spowrotem do klienta.
* Waiting - Liczba wolnych workerów, które są w stanie obsłużyć przychodzące rządania.

Więcej w [oficjalnej dokumentacji](http://nginx.org/libxslt/en/docs/http/ngx_http_stub_status_module.html)

* `Active connections`, `reading`, `writing` i `waiting` - te wartości są widoczne w logach.
  
##### Jak to rozumieć?

1) Najważniejsze jest niedopuszczenie aby wartość **Waiting** aby osiągnęła wartość 0.
2) Drugim podejrzanym sygnałem, że coś jest nie tak jest to, że **accepts** jest większe od **handled**. 
Oznacza to, że brakuje nam wolnych workerów i i niektóre żądania zostały odrzucone.



# Konfiguracja PHP i PHP-FPM.

### Logi

##### Error log

Logi te zawierają komunikaty błędów. 

Aby skonfigurować logowanie błędów otwieramy plik: `/etc/php/php-fpm.conf` i ustawiamy wartość
dyrektywy error_log na `/var/log/php-fpm/www.pool.error.log`

```
error_log = /var/log/php-fpm/www.pool.error.log
```


##### Access log

Logi te zawierają wszystkie requesty jakie php-fpm odebrał z wszystkich źródeł.

Przygotowanie naszej konfiguracji php-fpm zaczniemy od edycji puli php-fpm. W tym celu 
otwieramy plik `/etc/php/php-fpm.d/www.conf` i szukamy lini w której ustawione jest dyrektywa
`access.format`. U mnie domyślnie jest zakomentowana i wygląda mniej więcej tak:

```
;access.format = "%R - %u %t \"%m %r%Q%q\" %s %f %{mili}d %{kilo}M %C%%"
```

Jak wszędzie zaczynamy używać formatu JSON. linia ta powinna wyglądać tak:
```
access.format =  '{"pool_name": "%n", "cpu": "%C", "time_to_serve_request": "%d", "REQUEST_METHOD": "%{REQUEST_METHOD}e", "HTTP_HOST": "%{HTTP_HOST}e", "memory_alocated": "%{kilo}M kb","script_file_name": "%f", "query_string": "%q", "request_uri": "%r", "remote_ip": "%R", "response_code": "%s", "date": "%t", "remote_user": "%u"}'
```
Niestety mi nie udało się tej lini rozdzielić w kilka, i nie wygląda to najładniej, ale działa.

Następnie musimy zadbać aby aby te wiadomości były logowane, więc ustawiamy dyrektywę `access.log`:

```
access.log = /var/log/php-fpm/www.pool.access.log
```

W tym samym pliku ustawiamy dodatkową dyrektywę, która spowoduje, że wszystkie błędy
jakie wystąpią w parserze PHP oraz wszelkie komunikaty od workerów zostaną przekierowane 
do naszego logu, a dokładniej do pliku z dyrektywy `error_log` w dyrektywie `[global]`

```
catch_workers_output = yes
```

##### Slow log

Slow log jest specyficznym logiem, ponieważ zawiera informacje tylko o requestach, które
wykonywały się zbyt długo. Jest to niezwykle przydatny log i powinniśmy dążyć aby był pusty.

Przystąpmy do konfiguracji logowania slow requestów.
Otwieramy nasz plik z ustawieniami puli `/etc/php/php-fpm.d/www.conf`, i ustawiamy dwie dyrektywy:

```
slowlog = /var/log/php-fpm/www.pool.slow.log
request_slowlog_timeout = 2s
```

Pierwsza dyrektywa(`slowlog`) mówi nam, gdzie będą logowane zmiany, druga(`request_slowlog_timeout`),
mówi jaki jest minimalny czas wykonania requestu, który zostanie zalogowany.



##### Ustawienia parsera PHP

Otwieramy plik `/etc/php/php-fpm.d/www.conf` i na końcu dodajemy następujące linie: 

```
php_flag[display_errors] = off
php_admin_value[error_log] = /var/log/php.error.log
php_admin_flag[log_errors] = on
```

Linie te nadpisują dyrektywy z pliku php.ini, więc tamte wartości zostaną zignotorowane.
Jeśli nie chcesz tego robić, to poprostu zignoruj tą sekcję, ale pamiętaj aby włączyć logowanie
do pliku w `/etc/php/php.ini`. 


##### Więcej? 

Po więcej informacji informacji zapraszam do [oficjalnej dokumentacji](http://php.net/manual/en/install.fpm.configuration.php).


### Metryki











## Linki
Wstep do PHP-FPM, Wstęp do Nginxa
[Documentacji Nginxa](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format)
[dokumentacji](http://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)
[oficjalnej dokumentacji](http://nginx.org/libxslt/en/docs/http/ngx_http_stub_status_module.html)





