# Exceptions

### Wstęp
Co a w zasadzie kto skłonił mnie do napisania tego artykułu? Tak naprawdę to znajomi którzy kilkukrotnie pytali o wyjątki. 

### Co to jest tak naprawdę ten wyjątek?
Dokumentacja PHP mówi:
>  An exception can be thrown, and caught ("catched") within PHP. Code may be surrounded in a try block, to facilitate the catching of potential exceptions.

> pl: "W PHP wyjątek może zostać rzucony i złapany. Kod może być objęty w blok try  by ułatwić wychwytywanie potencjalnych sytuacji wyjatkowych(czyt. "niechcianych).

No spoko, ale co to ten wyjątek jest? Chyba nigdzie nie znalazłem odpowiedzi na to pytanie...
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

I moglibyśmy wyliczać kolejne błędy, ale nie to jest celem tego artykułu. Jak widzimy możemy dając użytkownikowi program musimy przewidzieć masę niestandardowych rzeczy, do których może doprowadzić użytkownik korzystające z naszego programu. Właśnie w obsłudze takich rzeczy pomagają nam wyjątki.

##### Jak działają wyjątki?
Po pierwsze wyjątek musi zostać rzucony poprzez słowo kluczowe throw 
```php
throw new Exception("..."); 
```
w tym momencie tracimy program **przerywa** swoje działanie i wykonuje **skok** w inne miejsce. Jeśli wyjątek został rzucony poza blokiem "**try**" to program kończy się **błędem**. Natomiast jeśli rzuciliśmy wyjątek w kodzie który jest objęty klamrami bloku **try** { }, to program wykonuje skok do najbliższego bloku **catch** który **jest go w stanie złapać**. W momęcie złapania wyjątku **odzyskujemy kontrolę** nad programem i dostajemy dostęp do rzuconego wyjątku. Taki rzucony wyjątek może mieć w sobie informację o błędzie który wystąpił w programie. Najprostszymi informacjami niesionymi przez taki obiekt są: wiadomośc, kod błędu, plik w którym został wyrzucony, linię itp. Oczywiście sami możemy definiować jakie informacje ma taki obiekt.

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

```php
Class ArythmeticOperation
{
	public function (float

