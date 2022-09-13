⚠️ WIP ⚠️
# Symulacja stacji pogodowej - LabView/Arduino/PicSimLab 🇵🇱
## Wstęp
Program wykonany jako projekt zaliczeniowy na zajęcia laboratoryjne. Służy od do symulacji pracy stacji pogodowej. Posiada funkcję poboru danych z płytki Arduino (tu symulowanej przez PicSimLab), obliczeniu wartości średnich oraz wyświetlenia wartości na grafach i polach tekstowych. Stworzony program wykorzystuje połączenie ze sobą trzech oprogramowań: LabView, Arduino oraz PicSimLab. PicSimLab służy do symulacji sygnałów wejściowych. Arduino wykorzystano w celu połączenia ze sobą PicSimLab oraz LabView. Sam program wykonany został w labView.
## PicSimLab
Program PicSimLab służy do symulowania płytki Arduino bez posiadania fizycznego objektu. Jest to bardzo wygodne, ponieważ ten sprzęt niekoniecznie może być dostępny przy procesie projektowania, a w praktyce wystarczy do kanału wejściowego podłączyć rzeczywistą płytkę i reszta kodu działa jak należy. W naszym przypadku 'stacja pogodowa' będzie prowadziła pomiar temperatury, wilgotności powietrza, ciśnienia oraz zapylenia powietrza. Istnieje w PicSimLab komponent pomiarowy temperatury oraz wilgotności - jest nim sensor SHT3X. Fizyczny sensor wyprodukowała firma Sensirion, natomiast cyfrowa wersja tego czujnika składa się z biblioteki napisanej przez [Roba Tillaarta](https://github.com/RobTillaart/SHT31). Kolejną częścią jest pomiar ciśnienia oraz zapylenia. Niestety, w domyślnym pakiecie PicSimLab nie ma do nich czujnika. Ze względu na ograniczenia czasowe są one symulowane przez potencjometry.

![PicSimLab](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/picsimlab.png)

## Arduino
Jak wspomniano wcześniej, tutaj oprogramowanie Arduino użyto tylko jako pośrednik pomiędzy płytką/PicSimLab a LabView, gdzie znajduje się prawdziwy program. Z tego powodu kod Arduino jest prosty. Uruchamiamy połączenie Serial z prędkością przesyłu danych 9600 bit/s. Potem co cykl sczytujemy dane z wejść analogowych A0, A1, A2, A3. Sczytane wielkości wyświetlamy w jednej linii z opisami, a na koniec przechodzimy do kolejnej linii. Pomiędzy syklami mamy przerwę 1000 ms celem synchronizacji z LabView.

![Arduino](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/arduino.png)

## LabView
Większość programu wykonano w LabView. To tutaj mamy pełny kod wraz z interfejsem użytkownika. Podstawą programu jest maszyna stanu. Maszyna stanu w LabView składa się z trzech komponentów: pętli While, bloku Case oraz opcjonalnego bloku Event. Działanie maszyny stanu polega na tym, że program jest wykonywany w pętli co okres, wewnętrzne warunki przełączają aktualny stan programu a wewnątrz danego stanu możliwe jest wykrywanie akcji użytkownika. Przykładowo, ten program posiada trzy stany: Init, Password oraz Measurement. Init to inicjalizacja wartości początkowych. W LabView odbywa się to poprzez połączenie stanów wejściowych z poza pętli do wewnątrz. Stan Init służy po to, aby przed wykonaniem dalszych operacji dane wejściowe zostały sczytane. Jako dane wejściowe mamy początkowe wartości tablic godzinowych/minutowych dla każdej z mierzonych wielkości, port Arduino z którego pobieramy dane oraz stan Init. Tu również można dać początkową informację dla użytkownika. Ze względu na to że 'stacja pogodowa' zabezpieczona jest hasłem, dajemy informacje "Podaj hasło".

![Init](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/init.png)

W stanie Password mamy blok Event powiązany z naciśnięciem przycisku Potwierdz Haslo. Wewnątrz sprawdzane jest, czy wartość pola Haslo znadza się z rzeczywistym hasłem (tutaj akurat jako hasło wybrano słowo "Haslo", ale łatwo można go zmienić lub nawet ustawić hasło zewnętrznie). Jeśli hasło jest nieprawidłowe, wyświetlamy informację do użytkownika o tym i pozostajemy w stanie Password. Jeśli jest prawidłowe, przechodzimy do stanu Measurement.

![Password](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/password.png)

Stan Measurement reprezentuje całość operacyjną programu. To tutaj wykonujemy odczyt danych, skalujemy dane, obliczamy średnie wartości oraz wyświetlamy je w odpowiednich polach.

![Measurement](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/measurement.png)

Podążając za torem danych, wchodzimy do stanu Measurement oraz do wydarzenia Timeout do bloku VISA Open. Timeout służy do synchronizacji, prowadzi update co 10 ms. Z kolej blok VISA Open pozwala na odczytanie ze strumienia danych aktualnej linii wyjściowej ze źródła. Ta aktualna linia to ta sama linia co stworzyliśmy w poprzedznim dziale. Zawiera ona wszystkie watości zmierzone przez płytkę wraz z opisami. Z tego powodu musimy wykonać pewne filtrowanie. Chcemy odzyskać z łańcucha znaków liczbę pomiędzy oznaczeniami. W tym celu stosujemy podprogram ExtractValue. Bierze on wejściowy łańcuch, używa funkcji Substring do odfiltrowania potrzebnej wartości a na koniec zamienia z łańcucha na liczbę. Po odfiltrowaniu czterech mierzonych wielkości, przechodzimy do przeskalowania.

![ExtractValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/extract.png)

Przeskalowanie jest niezbędne ze względu na to, że czujniki cyfrowe wysyłają wartość cyfrową, a nie rzeczywistą zmierzoną. Posiadają one kompletnie inne skale ze względu na co nie da się ich bezpośrednio odczytać. Tutaj, przykładowo, pomiar temperatury posiada skalę cyfrową od 125 do 897, a fizyczny czujnik ma skalę od -40 °C do 125 °C. Do przeskalowania równiez używany podprogramu, RescaleValue. Działa on na wykonaniu prostego obliczenia: xn = smn + (sMn - smn)(xs - sms)/(sMs - sms), gdzie xn oraz xs to x nowy i stary, smn oraz sMn to minimum oraz maksimum nowej skali, sms oraz sMs to minimum oraz maksumym nowej skali. Wynik operacji można wyświetlić w polu aktualnej wartości.

![RescaleValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/rescale.png)

Przeskalowane wartości przechodzą są pakowane w jeden klaster oraz przekazywane do podprogramów MassAverages. Pozostałe wejścia tych podprogramów to klastry wejściowe do których wpisujemy obliczone średnie oraz wartość przez którą dzielimy. Chcemy, aby Operacja średniej działała nawet jeśli nie minęła minuta lub godzina potrzebna do prawdziwej średniej minutowej/godzinnej. Z tego powodu, jeśli ilość miniętych iteracji nie przekracza 60 lub 3600 to przekazujamy aktualną ilość sekund. Jeśli tak, to przenosimy 60 lub 3600. Srednie obliczane są w środku. MassAverages to tylko czterokrotne wywołanie kolejnego podprogramu - AverageValue.

![MassAverages](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/mass.png)

Obliczanie wartości średniej jest nieintuicyjne. Gdybyśmy mieli wielkości stałe, wystarczyłoby zrobić sumę elementów tablicy i podzielić przez ilość. Tutaj jednak mamy wartości dynamiczne. Co cykl zmieniają się elementy tablicy. Z tego powodu musimy wykonać kilka dodatkowych operacji. Należy usunąć ostatni element, przesuwając wszystkie elementy tablicy o jeden a potem jako pierwszy element dodać aktualne dynamiczne wejście. Dopiero potem da się wykonać średnią. Ten podprogram działa tylko i wyłącznie jeśli znajduje się wewnątrz pętli While wraz z rejestrem przesuwnym dla tablicy.

![AverageValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/average.png)

Po obliczeniu wartości średnich, można je wyświetlić w polach oraz na grafach. Tym kończy się wydarzenie Timeout. Poza Timeout mamy również dwa wydarzenia dodatkowe: "Wyszysc Wykresy" oraz "Zapisz Wykresy". Odpowiadają one przyciskom o tych samych nazwach. "Wyczysc Wykresy" polega na tym, że do historii wykresów wpisujemy zera, aby usunąć poprzednie pomiary. "Zapisz Wykresy" to zapisanie aktualnych wykresów jako bitmapy. Przydatne jest to w celu przyszłych badań. Wykorzystuje ono wywołanie ExportImage, gdzie można ustalić folder wyjściowy oraz nazwę pliku.

![Clear](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/clear.png)

![Save](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/save.png)

## Program
Okno programu zawiera wspomniane wcześniej elementy. Można wybrać port do którego podłączona jest płytka/PicSimLab. W celach debugowania widać ilość wykonanych iteracji programu oraz iteracji pomiaru. Tuż pod tym wpisuje się hasło. Główna część programu wyświetla się mierzone wielkości aktualne oraz średnie i w formie tekstowej i na wykresach.

![Program](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/working.png)

Zapisane wykresy widoczne poniżej:

![T](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/T.bmp)

![RH](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/RH.bmp)

![p](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/p.bmp)

![PH10](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/PH10.bmp)

Łatwo sprawdzić że program działa prawidłowo zmieniając wartości na płytce/PicSimLab oraz obserwując zmiany stanu w programie. Całość szczegółowego opisu programu dostępna w [moim projekcie](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/Kacper%20Hoffman%20-%20Projekt%202.pdf).

---
# Weather station simulation - LabView/Arduino/PicSimLab 🇬🇧
