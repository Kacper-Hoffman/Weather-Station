# Symulacja stacji pogodowej - LabView/Arduino/PicSimLab 🇵🇱
## Wstęp
Program wykonany jako projekt zaliczeniowy na zajęcia laboratoryjne. Służy od do symulacji pracy stacji pogodowej. Posiada funkcję poboru danych z płytki Arduino (tu symulowanej przez PicSimLab), obliczeniu wartości średnich oraz wyświetlenia wartości na grafach i polach tekstowych. Stworzony program wykorzystuje połączenie ze sobą trzech oprogramowań: LabView, Arduino oraz PicSimLab. PicSimLab służy do symulacji sygnałów wejściowych. Arduino wykorzystano w celu połączenia ze sobą PicSimLab oraz LabView. Sam program wykonany został w labView.
## PicSimLab
Program PicSimLab służy do symulowania płytki Arduino bez posiadania fizycznego objektu. Jest to bardzo wygodne, ponieważ ten sprzęt niekoniecznie może być dostępny przy procesie projektowania, a w praktyce wystarczy do kanału wejściowego podłączyć rzeczywistą płytkę i reszta kodu działa jak należy. W naszym przypadku 'stacja pogodowa' będzie prowadziła pomiar temperatury, wilgotności powietrza, ciśnienia oraz zapylenia powietrza. Istnieje w PicSimLab komponent pomiarowy temperatury oraz wilgotności - jest nim sensor SHT3X. Fizyczny sensor wyprodukowała firma Sensirion, natomiast cyfrowa wersja tego czujnika składa się z biblioteki napisanej przez [Roba Tillaarta](https://github.com/RobTillaart/SHT31). Kolejną częścią jest pomiar ciśnienia oraz zapylenia. Niestety, w domyślnym pakiecie PicSimLab nie ma do nich czujnika. Ze względu na ograniczenia czasowe są one symulowane przez potencjometry.

![PicSimLab](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/picsimlab.png)

## Arduino
Jak wspomniano wcześniej, tutaj oprogramowanie Arduino użyto tylko jako pośrednik pomiędzy płytką/PicSimLab a LabView, gdzie znajduje się prawdziwy program. Z tego powodu kod Arduino jest prosty. Uruchamiamy połączenie Serial z prędkością przesyłu danych 9600 bit/s. Potem co cykl sczytujemy dane z wejść analogowych A0, A1, A2, A3. Sczytane wielkości wyświetlamy w jednej linii z opisami, a na koniec przechodzimy do kolejnej linii. Pomiędzy cyklami mamy przerwę 1000 ms celem synchronizacji z LabView.

![Arduino](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/arduino.png)

## LabView
Większość programu wykonano w LabView. To tutaj mamy pełny kod wraz z interfejsem użytkownika. Podstawą programu jest maszyna stanu. Maszyna stanu w LabView składa się z trzech komponentów: pętli While, bloku Case oraz opcjonalnego bloku Event. Działanie maszyny stanu polega na tym, że program jest wykonywany w pętli co okres, wewnętrzne warunki przełączają aktualny stan programu a wewnątrz danego stanu możliwe jest wykrywanie akcji użytkownika. Przykładowo, ten program posiada trzy stany: Init, Password oraz Measurement. Init to inicjalizacja wartości początkowych. W LabView odbywa się to poprzez połączenie stanów wejściowych z poza pętli do wewnątrz. Stan Init służy po to, aby przed wykonaniem dalszych operacji dane wejściowe zostały sczytane. Jako dane wejściowe mamy początkowe wartości tablic godzinowych/minutowych dla każdej z mierzonych wielkości, port Arduino z którego pobieramy dane oraz stan Init. Tu również można dać początkową informację dla użytkownika. Ze względu na to że 'stacja pogodowa' zabezpieczona jest hasłem, dajemy informacje "Podaj hasło".

![Init](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/init.png)

W stanie Password mamy blok Event powiązany z naciśnięciem przycisku Potwierdz Haslo. Wewnątrz sprawdzane jest, czy wartość pola Haslo znadza się z rzeczywistym hasłem (tutaj akurat jako hasło wybrano słowo "Haslo", ale łatwo można go zmienić lub nawet ustawić hasło zewnętrznie). Jeśli hasło jest nieprawidłowe, wyświetlamy informację do użytkownika o tym i pozostajemy w stanie Password. Jeśli jest prawidłowe, przechodzimy do stanu Measurement.

![Password](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/password.png)

Stan Measurement reprezentuje całość operacyjną programu. To tutaj wykonujemy odczyt danych, skalujemy dane, obliczamy średnie wartości oraz wyświetlamy je w odpowiednich polach.

![Measurement](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/measurement.png)

Podążając za torem danych, wchodzimy do stanu Measurement oraz do wydarzenia Timeout do bloku VISA Open. Timeout służy do synchronizacji, prowadzi update co 10 ms. Z kolej blok VISA Open pozwala na odczytanie ze strumienia danych aktualnej linii wyjściowej ze źródła. Ta aktualna linia to ta sama linia co stworzyliśmy w poprzednim dziale. Zawiera ona wszystkie watości zmierzone przez płytkę wraz z opisami. Z tego powodu musimy wykonać pewne filtrowanie. Chcemy odzyskać z łańcucha znaków liczbę pomiędzy oznaczeniami. W tym celu stosujemy podprogram ExtractValue. Bierze on wejściowy łańcuch, używa funkcji Substring do odfiltrowania potrzebnej wartości a na koniec zamienia z łańcucha na liczbę. Po odfiltrowaniu czterech mierzonych wielkości, przechodzimy do przeskalowania.

![ExtractValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/extract.png)

Przeskalowanie jest niezbędne ze względu na to, że czujniki cyfrowe wysyłają wartość cyfrową, a nie rzeczywistą zmierzoną. Posiadają one kompletnie inne skale ze względu na co nie da się ich bezpośrednio odczytać. Tutaj, przykładowo, pomiar temperatury posiada skalę cyfrową od 125 do 897, a fizyczny czujnik ma skalę od -40 °C do 125 °C. Do przeskalowania równiez używany podprogramu, RescaleValue. Działa on na wykonaniu prostego obliczenia: xn = smn + (sMn - smn)(xs - sms)/(sMs - sms), gdzie xn oraz xs to x nowy i stary, smn oraz sMn to minimum oraz maksimum nowej skali, sms oraz sMs to minimum oraz maksimum starej skali. Wynik operacji można wyświetlić w polu aktualnej wartości.

![RescaleValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/rescale.png)

Przeskalowane wartości przechodzą są pakowane w jeden klaster oraz przekazywane do podprogramów MassAverages. Pozostałe wejścia tych podprogramów to klastry wejściowe do których wpisujemy obliczone średnie oraz wartość przez którą dzielimy. Chcemy, aby operacja średniej działała nawet jeśli nie minęła minuta lub godzina potrzebna do prawdziwej średniej minutowej/godzinnej. Z tego powodu, jeśli ilość miniętych iteracji nie przekracza 60 lub 3600 to przekazujamy aktualną ilość sekund. Jeśli tak, to przenosimy 60 lub 3600. Srednie obliczane są w środku. MassAverages to tylko czterokrotne wywołanie kolejnego podprogramu - AverageValue.

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
## Introduction
Program created as the final project of laboratory classes. It's used to simulate a weather station. It has the ability to read input from an Arduino board (simulated with PicSimLab), calculating average values and displaying then both on graphs and text fields. It makes use of three different programs: LabView, Arduino and PicSimLab. PicSimLab is used to simulate the input signals. Arduino is used to connect PicSimLab with LabView. The program itself was written in labView.
## PicSimLab
PicSimLab is used to simulate an Arduino board without having the physical board. This is very useful, because a physical board is not always available during design phase. At any point the input channel can be switched to the physical board and the code will still work. In our case the 'weather station' will measire temperature, humidity, pressure and air pollution. PicSimLab has a component used for measuring temperature and humidity - the SHT3X sensor. The physical sensor was produced by Sensirion, but the digital library for it has been written by [Rob Tillaart](https://github.com/RobTillaart/SHT31). The next part is the measurement of pressure ans pollution. Unfortunately, PicSimLab doesn't contain a built-int sensor for them. Tue to deadlines they have been simulated with potentiometers.

![PicSimLab](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/picsimlab.png)

## Arduino
As mentioned before, Arduino software was only used as a mediary between the board/PicSimLab and LabView, where we have the real program. Because of this the Arduino code is simple. We start up the Serial connection with a speed of 9600 bit/s. Then every cycle we read the values of analogue inputs A0, A1, A2, A3. Read values are written to the output along with labels in one line and then the next line is started. Between cycles we have 1000 ms delay for synchronization with LabView.

![Arduino](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/arduino.png)

## LabView
Most of the program was made in LabView. Here we have the full code along with the interface. The basis of our program is the state machine. State machine in LabView is based on three components: While loop, Case block and an optional Event block. The state machine works by having the loop execute the code every cycle, according to different conditions the state changes and the program detects the user actions with Event blocks. For example, this program contains three states: Init, Password and Measurement. Init is the initialization of initial values. In LabView is done by connecting the initial values from outside the loop into the loop. Init state is used, so that the initial values are read at least once before executing the rest of the code. As input values we have the tables of minute/hour averages, the Arduino Port and the Init state. Here we can also give an initial information for the user. Because our 'weather station' is password protected, we inform the user to give the password through "Podaj hasło".

![Init](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/init.png)

In the Password state we have the Event block connected to clocking the "Potwierdz Haslo" (Confirm Password) button. Here we check whether the input password is the same as the set password (here as password we literally use the word "Password" as "Haslo", but this can be easly changed or even set externally). If the password is incorrect we inform the user and retain the Password state. If it's correct we proceed to the Measurement state.

![Password](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/password.png)

The Measurement state represents the operational whole of the program. Here we read data, calculate averages and display them in appropriate fields.

![Measurement](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/measurement.png)

Following thestream of data, we enter the Measurement state and through the Timeout event into the VISA Open block. Timeout is used for Synchronisation, it updates every 10 ms. VISA Open is used to read the current output line of the Arduino port. This is the same line as the one we created in the previous chapter. It contains all the measured values along with labels. Because of this we need to do some filtering. We want to recover the values from the character string. To do this we use the ExtractValue subprogram. It takes the input string, uses Substring to filter out the value and then converts the string to number. After filtering we move on to rescaling.

![ExtractValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/extract.png)

Rescaling is necessary, because the input value is a 'digital' value, not the real measured one. They have completely different scales, so they cannot be directly read. Here for example, temperature measurement has a scale from 125 to 897, while the physical sensor has a scale from -40 °C to 125 °C. To rescale we also use a subprogram, RescaleValue. It's based on a simple calculation: xn = smn + (sMn - smn)(xs - sms)/(sMs - sms), where xn and xs are the new and old x, smn and sMn are th minimum and maximum of the new scale, sms and sMs are the minimum and maximum of the the old scale. The results of the operation can be displayed in the appropriate fields.

![RescaleValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/rescale.png)

Rescaled values are packed into a single cluster and used as input for the MassAverages subprogram. The other inouts are the tables of minute and hour averages as well as the value we divide by. We want the average operation to work even if the ull minute or hour haven't passed yet. Because of this, if the number of iterations hasn't passed 60 or 3600, we input the current number of seconds. If it has, we input 60 or 3600. The averages are calculated inside. MassAverages is just a four time call of the next subprogram - AverageValue.

![MassAverages](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/mass.png)

Calculating average values is unintuitive. If we had static values, we'd simply sum up the table and divide by number of elements. Here however we have a dynamic input. Every cycle the table contents change. Because of this we have to preform more calculations. We have to delete the last element, rotating the entire table by one and placing the current 'runnink input' as the first element. Only then can we calculate the average. This subprogram can work only if it's inside a While loop with a shift register for the table.

![AverageValue](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/average.png)

aFter calculating the average values, we can display them in graphs and text fields. This ends the Timeout Event. Besides Timeout we have two more events: "Wyszysc Wykresy" (Clear Graphs) and "Zapisz Wykresy" (Save Graphs). They represent the buttons with the same names. "Wyczysc Wykresy" works by inputting zeros into the history of graphs, so that their history is cleared. "Zapisz Wykresy" saves graphs as bitmaps. This can be used for future tests. It uses ExportImage, where we can set the output location and file name.

![Clear](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/clear.png)

![Save](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/save.png)

## Program
The program window contains previously mentioned elements. We can set the port where we have connected the board/PicSimLab. For debug purposes we have the number of program and measurement iterations. Below this we input the password.The main part of the program displays the current and average values in text fields and graphs.

![Program](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/working.png)

Saved graphs belov:

![T](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/T.bmp)

![RH](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/RH.bmp)

![p](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/p.bmp)

![PH10](https://raw.githubusercontent.com/Kacper-Hoffman/Weather-Station/main/PH10.bmp)

It's easy to test that the program works by changin inputs on the board/PicSimLab and observe the program's reactions. The full, detailed description of the program is available in [my project (🇵🇱 only)](https://github.com/Kacper-Hoffman/Weather-Station/blob/main/Kacper%20Hoffman%20-%20Projekt%202.pdf).
