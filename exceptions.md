# Exceptions

### Wstęp
Co, a w zasadzie kto skłonił mnie do napisania tego artykułu? Tak naprawdę to znajomi którzy kilkukrotnie pytali o wyjątki. 

### Co to jest tak naprawdę ten wyjątek?
Dokumentacja PHP mówi:
>  An exception can be thrown, and caught ("catched") within PHP. Code may be surrounded in a try block, to facilitate the catching of potential exceptions.

> pl: "W PHP wyjątek może zostać rzucony i złapany. Kod może być objęty w blok try  by ułatwić wychwytywanie potencjalnych sytuacji wyjatkowych(czyt. "niechcianych).

No spoko, ale czym ten wyjątek jest? Chyba nigdzie nie znalazłem odpowiedzi na to pytanie...
Wyjątek to nic innego jak zwykła klasa wbudowana w język programowania, która posiada w sobie jakieś predefiniowane metody oraz własności(zmienne). Klasy wyjątków przeważnie mogą zostać rozbudowane (o ile nie są finalne). Rozszeżać taką klasę można jak każdą inną poprzez dziedziczenie. Jak mówi dokumentacja klasy "wyjątków" mogą być "rzucane" i "łapane" jest to specjalna funkcjonalność klas z rodziny Exception.

### Po co nam to? Jak to działa? Co to jest to "rzucanie" i "łapanie"
Nie bez powodu czynności które możemy wykonywać z wyjątkami nazwyają się "rzucaniem" i "łapaniem". Tak jak w normalnym świecie gdy rzucimy na przykład piłkę możemy ją złapać aby rzucić ją kolejny raz albo poprostu odłożyć na swoje miejsce. W językach programowania zachodzi pewna analogia, którą postaram się niżej wyjaśnić.

###### Po co są wyjątki?
Tak naprawdę jest to jedno z pytań na które niema jednoznacznej odpowiedzi, ale ogólnie pomagają nam one sterować aplikacją, wyłapywać błędy, które wcześniej przewidzieliśmy, że mogą wystąpić.
Prosty przykład:
> **Zadanie:** Napisz program który będzie analizował dokumenty XML, szukał w nich wszystkich numerów telefonu i zwracał nam je.

Mamy zadanie, więc napiszmy sobie algorytm:
```
1. Otwórz plik XML.
2. Otwórz główny węzeł i zacznij przetwarzać po kolei wszystkie węzły znajdujące się w nim.
3. Dla każdego węzła spradź czy zawiera numer telefonu
    - Jeśli tak to zapisz do tablicy.
    - Jeśli nie to pomiń aktualny węzeł
4. Zwróć tablicę z znalezionymi numerami.
```
Oczywiście jest to bardzo uproszczony algorytm, ale i tutaj mogą pojawić się sytuacje których nie chcemy. Przeanalizujmy sobie punkt po punkcie nasz algorytm:
1. Otwórz plik XML.
    - Tutaj mogą pojawić się problemy w postaci: **nieistniejącego pliku** oraz tego, że wczytany plik **nie jest poprawny plikiem XML**. Jakie naspstwa mają takie błędy? 
        - Jeśli spróbujemy odczytać plik który nie istnieje to nie zostanie utworzony obiekt "uchwytu pliku" i będzie on NULL'em co spowoduje błędy w dalszym działaniu programu. 
        - Jeśli plik nie jest plikiem XML to nie utworzymy obiektu Parsera przez co też będziemy operowali na pustym wskaźniku.

2. Otwórz główny węzeł i zacznij przetwarzać po kolei wszystkie węzły znajdujące się w nim.
  - Ale co gdy dokument jest pusty i **główny węzeł nie istnieje**? Gdy nie wczytamy węzła i zaczniemy sprawdzać czy ma on dzieci? Znów będziemy chcieli operować na pustym wskaźniku.

I moglibyśmy wyliczać kolejne błędy, ale nie to jest celem tego artykułu. Jak widzimy dając użytkownikowi program musimy przewidzieć masę niestandardowych rzeczy, do których może doprowadzić użytkownik korzystając z naszego programu. Właśnie w obsłudze takich rzeczy pomagają nam wyjątki.

##### Jak działają wyjątki?
Po pierwsze wyjątek musi zostać rzucony poprzez słowo kluczowe throw 
```php
throw new Exception("..."); 
```
w tym momencie interpreter **przerywa** swoje działanie i wykonuje **skok** w inne miejsce. Jeśli wyjątek został rzucony poza blokiem "**try**" to program kończy się **błędem**. Natomiast jeśli rzuciliśmy wyjątek w kodzie który jest objęty klamrami bloku **try** { }, to program wykonuje skok do najbliższego bloku **catch** który **jest go w stanie złapać**. W momęcie złapania wyjątku **odzyskujemy kontrolę** nad programem i dostajemy dostęp do rzuconego wyjątku. Taki rzucony wyjątek może mieć w sobie informację o błędzie który wystąpił w programie. Najprostszymi informacjami niesionymi przez taki obiekt są: wiadomośc, kod błędu, plik w którym został wyrzucony, linię itp. Oczywiście sami możemy definiować jakie informacje ma taki obiekt.

>> Analizując poniższy przykład zastanów się które działania się wykonają po kolei i jaka jest wartość zmiennej $i po wykonaniu programu.
Przykład:
```php
try {
                try {
                    //Dzialanie 1:
                    $i = 0;
                    //Dzialanie 2:
                    $i = $i + 10; //Do tego momentu kod się wykonuje normalnie

                    throw new RuntimeException("Komunikat"); //Tutaj po rzuceniu wyjątku następuje skok do najbliższego bloku catch który jest w stanie złapać ten wyjątek

                    //Dzialanie 3:
                    $i = $i + 5; //Ten kod się nigdy nie wykona, ponieważ został przeskoczony przez wyrzucenie wyjątku
                } catch (LogicException $ex) {
                    print('Wystąpił wyjątek aplikacji: '.$ex->getMessage()); //Ten komunikat nie zostaje wyświetlony ponieważ my rzuciliśmy RuntimeException a ten blok obsługuje wyjątek logiczny
                }
                
                //Dzialanie 4:
                $i = $i + 2; //Ten kod też nie zostanie wykonany bo wyjątek nie został złapany więc próbuje skakać dalej do kolejnego bloku catch
            } catch (RuntimeException $ex) { //W tym miejscu nasz wyjątek który rzuciliśmy zostaje złapany i wykona się kod z tego bloku
                //W tym momencie odzyskujemy pełną kontrolę nad działaniem programu
                print('Wystąpił wyjątek aplikacji typu Runtime: '.$ex->getMessage()); //Na ekranie zobaczymy ten komunikat
            }

            //Dzialanie 5:
            $i = $i + 20; //Ten kod się wykona się już normalnie
```

Wykonają się działania: 1, 2, 5, więc zmienna $i = $i + 10 + 20. Wiec po wykonaniu programu $i = 30


Przykład:
> **Zadanie:** Napisz program który umożliwi dzielenie dwóch liczb.

Zadanie bardzo proste ale wystarczy do zobrazowania zasady działania wyjątków

Sama funkcjonalność prezentuje się następująco
```php
Class Calculator
{
	public function div(float $a, float $b): float
    {
        return $a / $b;
    }
}
```
Kod jest raczej prosty, zawiera jedną metodę która wykonuje dzielenie, ale rozważmy co się stanie jeśli wykonamy taki kod:

```php
$calc = new Calculator();
$result =  $calc->div(10, 0);
echo 'Wynik dzielenia 10 / 0 to: '.$result;
```

Otrzymamy błąd który nie wygląda przyjemnie, dodatkowo powyższe działanie powoduje niepoprawne działanie naszej aplikacji.
```
Warning: Division by zero in calculator.php on line 7 
Wynik dzielenia 10 / 0 to:
```

A co jeśli podamy argument większy niż możemy?
```php
$calc = new Calculator();
$result =  $calc->div(PHP_INT_MAX*PHP_INT_MAX, 2);
echo 'Wynik dzielenia '.(PHP_INT_MAX*PHP_INT_MAX).' / 0 to: '.$result;
```
W takim wypadku otrzymamy mega niedokładny wynik, wynika to z tego, że język PHP, gdy przepełnimy wartość zmiennej INT, przekształca ją do zmiennej typu float, zapisanej w postaci wykładniczej.
Oto co otrzymamy na ekranie:
```
Wynik dzielenia 8.5070591730235E+37 / 0 to: 4.2535295865117E+37
```
Oczywiście wynik jest bardzo niedokładny, nie chcemy aby potem użytkownicy naszego kalkulatora się skarżyli, więc zablokujemy tą funkcjonalnośc, opcjonalnie w przyszłości możemy dodać w to miejsce jakiś algorytm pozwalający na dzialania na tak ogromnych liczbach.


I tutaj możemy to rozwiązać na dwa sposoby mniej elegancki manualny, ale z pomocą przychodza nam wyjątki.
Pierwszy sposób:
```php
Class Calculator
{
    public $error;


    public function div(float $a, float $b): ?float
    {
        if ($a > PHP_INT_MAX || $b > PHP_INT_MAX) {
            $this->error = 'Wpisane wartości są zbyt duże';
            return null;
        }

        if ($b === 0.0) {
            $this->error = 'Dzielenie przez 0 jest zabronione';
            return null;
        }

        return $a / $b;
    }
}



$calc = new Calculator();
$result =  $calc->div(PHP_INT_MAX*PHP_INT_MAX, 2);

if ($result !== null) {
    echo 'Wynik dzielenia '.(PHP_INT_MAX*PHP_INT_MAX).' / 0 to: '.$result;    
} else {
    echo 'Podczas działania kalkulatora wystąpił błąd: '.$calc->error;
}
```
Dokonaliśmy kilku modyfikacji kodu. Po pierwsze teraz w klasie kalkulatora przchowujemy błąd oraz podczas wystąpienia błędu zwracany jest null.
Aby funkcja pozwoliła nam zwrócić wartośc null musimy dodać znak zapytania przed zwracanym typem.

Jednak nie mamy informacji w której lini wystąpił błąd, w przypadku gdybyśmy chcieli zalogować błąd do logów. 

Aby zapewnić taką funkcjonalność możem skorzystać z wyjątków, zatem napiszmy prosty program:


```
Class Calculator
{
    public function div(float $a, float $b): float
    {
        if ($a > PHP_INT_MAX || $b > PHP_INT_MAX) {
            throw new Exception('Wpisane wartości są zbyt duże', 10);
        }

        if ($b === 0.0) {
            throw new Exception('Dzielenie przez 0 jest zabronione', 11);
        }

        return $a / $b;
    }
}



$calc = new Calculator();
try {
    $result =  $calc->div(PHP_INT_MAX*PHP_INT_MAX, 2);
    echo 'Wynik dzielenia '.(PHP_INT_MAX*PHP_INT_MAX).' / 0 to: '.$result; 
} catch(Exception $ex) {
    echo 'Podczas działania kalkulatora wystąpił błąd: '.$ex->getMessage();
    echo PHP_EOL;
    echo sprintf(
        'Błąd wystąpił w pliku: %s, w lini %d, kod błędu: %d',
        $ex->getFile(),
        $ex->getLine(),
        $ex->getCode()
    );
}
```

Oto wynik działania:
```
Podczas działania kalkulatora wystąpił błąd: Wpisane wartości są zbyt duże 
Błąd wystąpił w pliku: calculator, w lini 6, kod błędu: 10
```

Co zyskujemy korzystając z wyjątków poza tymi dodatkowymi informacjami?
Pozbywamy się if'ów, kod staje się prostszy, pozwala dokładnie sterowac przepływem informacji w naszym programie.

W naszym przykładzie skorzystaliśmy z wbudowanej w język klasy Exception, co nie jest najlepszym przykładem. 
Rozbudujmy nasz program o jakąś klasę wyświetlającą wynik na ekranie.




Oto nasz trochę bardziej rozbudowany kod:
```php
<?php
Class RendererException extends Exception
{
    const UNDEFINED_WINDOW_CODE = 21;
    
    public static function forUndefinedWindow(Throwable $prev = null) 
    {
        return new self('Nie można wyrenderować. Obiekt okna jest pusty', self::UNDEFINED_WINDOW_CODE, $prev);
    }    
}

Class CalculatorException extends Exception
{
    const UNDEFINED_WINDOW_CODE = 0;
    
    public static function forDivisionByZero(Throwable $prev = null) 
    {
        return new self('Dzielenie przez 0 jest zabronione', self::UNDEFINED_WINDOW_CODE, $prev);
    }
    
    public static function forArgumentOverflow(Throwable $prev = null) 
    {
        return new self('Wpisane wartości są zbyt duże', self::UNDEFINED_WINDOW_CODE, $prev);
    } 
}



Class Renderer
{
    private $window;
    
    public function renderNotification() {
        //....
        if ($this->window === null) {
            throw RendererException::forUndefinedWindow();
        }
        
        //.... Wyświetlanie na ekranie
    }
}


Class Calculator
{
    public function div(float $a, float $b): float
    {
        if ($a > PHP_INT_MAX || $b > PHP_INT_MAX) {
            throw CalculatorException::forArgumentOverflow();
        }

        if ($b === 0.0) {
            throw CalculatorException::forDivisionByZero();
        }

        return $a / $b;
    }
}



$calc = new Calculator();
try {
    $renrerer = new Renderer();
    
    
    $result =  $calc->div(PHP_INT_MAX*PHP_INT_MAX, 2);
    $renderer->renderNotification('Wynik dzielenia '.(PHP_INT_MAX*PHP_INT_MAX).' / 0 to: '.$result); 
} catch(CalculatorException $ex) {
    echo 'Podczas działania kalkulatora wystąpił błąd: '.$ex->getMessage();
    echo PHP_EOL;
    echo sprintf(
        'Błąd wystąpił w pliku: %s, w lini %d, kod błędu: %d',
        $ex->getFile(),
        $ex->getLine(),
        $ex->getCode()
    );
} catch (RendererException $ex) {
    echo 'Podczas renderowania obrazu wystąpił błąd: '.$ex->getMessage();
}
```

Dodaliśmy tutaj 3 nowe klasy:
    - **Renderer** - klasa która odpowiada za wyświetlanie okienek, nie musimy wiedzieć jak działa, ważne, że rzuca wyjątek gdy stanie się coś niewłaściwego.
    - **RendererException** - klasa która dziedziczy po Exception i można nią rzucać. Klasa będzie rzucana gdy wystąpi błąd podczas renderowania okienek naszej aplikacji.
    - **CalculatorException** - klasa która będzie rzucana podczas wystąpienia błędu w czasie obliczania.

Można tutaj zauważyć znaczącą różnice, dlaczego nie rzucać wyjątków klasy Exception, po co tworzyć nowe klasy?
Jest to trafne pytanie. Otóż lepiej stworzyć osobne klasy bo poprawia nam to organizację kodu w naszej aplikacji. Pozwala lepiej zorganizować przepływ informacji i ułatwić modfikację kodu. Dodatkowo jedne wyjątki możemy łapać w jednym miejscu drugie w drugim...

Drugą ważną rzeczą, jest to, że klasy wyjątków posiadają same metody statyczne. Robimy to po to, aby rodzielić implementację od obsługi błędów, dodatkowo poprawia nam to czytelność, bo co jeśli nasz kod będzie czytał np. programista z francji to który fragment będzie dla niego bardziej zrozumiały:

```php
if ($a > PHP_INT_MAX || $b > PHP_INT_MAX) {
    throw CalculatorException::forArgumentOverflow();
}
```
czy może:

```php
if ($a > PHP_INT_MAX || $b > PHP_INT_MAX) {
    throw new Exception('Wpisane wartości są zbyt duże', 10);
}
```


Oczywiście, że ten pierwszy. W tym trywialnym przykładzie mozna oczywiści odczytać z warunków dlaczego wyjątek jest rzucany, ale nie zawsze tak będzie. Poza tym izoluje nam to ładnie moduły. Dodatkowo wszystkei komunikaty mamy w jednym miejscu, oraz widzimy w jakich wypadkach jest rzucany konkretny wyjątek. 




Ostatnią ważną rzeczą jest to, ze wyjątki można rzucać w łańcuchu, mamy tym samym możliwość śledzenai jak po kolei występowały błędy w naszej aplikacji.
```php
<?php
try {
    throw new Exception('Msg1', 10); //1
} catch (Exception $ex) {
    try {
        throw new Exception('Msg2', 10, $ex); //2
    } catch (Exception $ex1) {
        try {
            throw new Exception('Msg3', 10, $ex1); //3
        } catch (Exception $ex2) {
            
            $tmpEx = $ex2; //4
            do {
                echo 'Wyjątek: '.$tmpEx->getMessage().PHP_EOL;
                $tmpEx = $tmpEx->getPrevious();
                
            } while($tmpEx !== null);
        }
    }
}
```

Wyjaśnijmy ten kod:
Po koleji są rzucane 3 wyjątki w kolejności 1, 2, 3. Dodatkowo podczas rzucania wyjątku 2 i 3 przekazywane są poprzednie wyjątki. W miejscu oznaczonym 4 jest prosta pętla do wyświetlania wszystkich rzuconych wyjątków. Wyjątki oczywiście wyświetlimy w kolejności odwrotnej.

Oto wynik programu:
```
Wyjątek: Msg3 
Wyjątek: Msg2 
Wyjątek: Msg1
```

Konstrukcja często jest stosowana aby określić skąd się wziął konkretny błąd, co go spowodowało.


Dodatkowym źródłem informacji jest dokumentacja PHP:
[Exceptions](http://php.net/manual/en/language.exceptions.php)
[Extending Exceptions](http://php.net/manual/en/language.exceptions.extending.php)
[Errors](http://php.net/manual/en/language.errors.php7.php)



