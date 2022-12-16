# Асписов Дмитрий Алексеевич, БПИ-218, ИДЗ 4, Вариант 29.

## Задание

На седых склонах Гималаев стоит древний буддистский монастырь: Гуань-Инь-Янь. Каждый год в день сошествия на землю боддисатвы Монахи монастыря собираются на совместное празднество и
показывают свое совершенствование на Пути Кулака. Всех соревнующихся
монахов разбивают на пары, победители пар бьются затем между собой и так
далее, до финального поединка. Монах который победил в финальном бою,
забирает себе на хранение статую боддисатвы. Реализовать многопоточное
приложение, определяющего победителя. В качестве входных данных используется массив, в котором хранится количество энергии Ци каждого монаха. При победе монах забирает энергию Ци своего противника. Разбивка на
пары перед каждым сражением осуществляется случайным образом. Монах,
оставшийся без пары, удваивает свою энергию, отдохнув от поединка. При
решении использовать принцип дихотомии.
 
 ## Решение

### Исходный код на языке С++ на 8 баллов:
```cpp
#include <algorithm>
#include <iostream>
#include <random>
#include <pthread.h>
#include <vector>
#include <fstream>
#include <utility>

// поток вывода в файл
std::ofstream ofs;
// мьютекс для дуэлей
pthread_mutex_t mutex;

// монах, который имеет свой id и значение силы (power)
struct monk {
    int id;
    int power;
};

// вектор с монахами
std::vector<monk> v;
// победители раунда
std::vector<monk> winners;

// функция считывает входные данные из консоли
void console_input() {
    std::cout << "Enter the amount of monks in your tournament:";

    size_t number_of_monks;
    std::cin >> number_of_monks;

    std::cout << "\nEnter power numbers of all the monks (\"3 4 10 6 7 8\"):";
    for (int i = 0; i < number_of_monks; i++) {
        monk m{};
        m.id = i + 1;
        std::cin >> m.power;

        v.push_back(m);
    }

    ofs.open("output.txt", std::ofstream::out | std::ofstream::trunc);
}

// функция случайным образом генерирует входные данные
void random_input() {
    // генератор случайных чисел
    std::random_device rd;
    std::mt19937 gen(rd());

    std::uniform_int_distribution<int> monks_amount_distribution(1, 100);
    std::uniform_int_distribution<int> power_distribution(0, 500);

    int monks_amount = monks_amount_distribution(gen);

    for (int i = 0; i < monks_amount; i++) {
        monk m{};
        m.id = i + 1;
        // генерация случайного значения сила от 0 до 500
        m.power = power_distribution(gen);
        v.push_back(m);
    }

    std::cout << "Monks that participate in the tournament: ";
    for (const monk &m : v) {
        std::cout << m.power << " ";
    }
    std::cout << "\n";

    ofs.open("output.txt", std::ofstream::out | std::ofstream::trunc);
}

// функция считывает данные из файла
void file_input() {
    std::string input_file, output_file;

    std::cout << "Enter an input file name:";
    std::cin >> input_file;

    std::cout << "Enter an output file name:";
    std::cin >> output_file;

    std::ifstream ifs(input_file);

    int number_of_monks;
    ifs >> number_of_monks;

    for (int i = 0; i < number_of_monks; i++) {
        monk m{};
        ifs >> m.power;
        m.id = i + 1;
        v.push_back(m);
    }

    ofs.open(output_file, std::ofstream::out | std::ofstream::trunc);
}

// функция проводит дэль между двумя монахами
void *duel(void *arg) {
    auto *fighters = static_cast<monk*> (arg);

    // определение победителя и проигравшего
    monk winner{}, loser{};
    winner = fighters[0];
    loser = fighters[1];
    if (winner.power < loser.power) {
        std::swap(winner, loser);
    }
    winner.power += loser.power;

    pthread_mutex_lock(&mutex);

    // вывод результатов дуэли
    std::cout << "Monk " << winner.id << " won a duel against monk " << loser.id
              << " and now has power of " << winner.power << "!\n";
    ofs << "Monk " << winner.id << " won a duel against monk " << loser.id
        << " and now has power of " << winner.power << "!\n";

    winners.push_back(winner);

    pthread_mutex_unlock(&mutex);

    return nullptr;
}

void run_fights() {
    // генератор чисел для перемешивания вектора с монахами
    std::random_device rd;
    std::mt19937 gen(rd());

    int round = 1;
    // запуск раундов дуэлей
    while (v.size() > 1) {
        // вывод номера раунда
        std::cout << "\n\t\t\tRound " << round << ":\n";
        ofs << "\n\t\t\tRound " << round << ":\n";
        round++;

        // перемешивание вектора монахов
        std::shuffle(v.begin(), v.end(), gen);

        // запуск параллельных сражений
        std::vector<pthread_t> threads;
        for (size_t i = 0; i < v.size() / 2; i++) {
            pthread_t thread;
            pthread_create(&thread, nullptr, duel, &v[2 * i]);
            threads.push_back(thread);
        }

        // ожидание окончания всех сражений
        for (pthread_t thread : threads) {
            pthread_join(thread, nullptr);
        }

        // сила монаха без пары удваивается
        if (v.size() % 2 == 1) {
            monk m = v[v.size() - 1];
            m.power *= 2;
            winners.push_back(m);
            std::cout << "Monk " << m.id << " now has power of " << m.power << "!" << std::endl;
            ofs << "Monk " << m.id << " now has power of " << m.power << "!" << std::endl;
        }

        // замена монахов победителями
        std::swap(v, winners);
        winners.clear();
    }

    // вывод данных о победителе турнира
    std::cout << "\nMonk " << v[0].id << " is the winner of the tournament!" << std::endl;
    ofs << "\nMonk " << v[0].id << " is the winner of the tournament!" << std::endl;
}

int main() {
    pthread_mutex_init(&mutex, nullptr);

    // выбор типа ввода
    int type;
    std::cout << "Choose the way you want to input monks for tournament:\n"
              << "\t\t1.Console\n" << "\t\t2.File\n" << "\t\t3.Random\n" << "Enter only a number:";
    std::cin >> type;
    switch (type) {
        case 1:
            console_input();
            break;
        case 2:
            file_input();
            break;
        case 3:
            random_input();
            break;
        default:
            std::cout << "Wrong argument!" << std::endl;
            return -1;
    }

    run_fights();

    pthread_mutex_destroy(&mutex);

    return 0;
}
```


### (4 балла) Модель параллельных вычислений, используемая при разработке многопоточной программы
При разработке программы использовался принцип дихотомии. 
Существует довольно очевидная теорема: "Если непрерывная функция на концах некоторого интервала имеет значения разных знаков, то внутри этого интервала у нее есть корень (как минимум, один, но м.б. и несколько)".
На базе этой теоремы построено численное нахождение приближенного значения корня функции. Обобщенно этот метод называется дихотомией, т.е. делением отрезка на две части.\
Источник: http://www.machinelearning.ru/wiki/index.php?title=%D0%9C%D0%B5%D1%82%D0%BE%D0%B4%D1%8B_%D0%B4%D0%B8%D1%85%D0%BE%D1%82%D0%BE%D0%BC%D0%B8%D0%B8 \
В моей задаче количество монахов с каждым раундом сокращалось вдвое, оставались лишь монахи победители. Поток получал на вход двух монахов, а возвращал одного.
### (4 балла) Входные данные программы, включающие вариативные диапазоны, возможные при многократных запусках
Целое число в диапазоне [1, 100] - количество монахов.\
Целое число в диапазоне [0, 500] - сила каждого из монахов.
### (4 балла) Консольное приложение, решающее поставленную задачу с использованием одного варианта синхропримитивов
Код представлен выше. Использовались мьютексы.

### (5 баллов) Сценарий, описывающий одновременное поведение представленных в условии задания сущностей в терминах предметной области
Субъекты: монахи. 
1) Монах сражается с другим монахом
2) Один из монахов побеждает
3) Монах получает нового соперника из числа монахов-победителей
Это повторяется, пока не останется один монах. И сражения всех монахов происходят одновременно.

### (6 баллов) Обобщенный алгоритм, используемый при реализации программы исходного словесного сценария
Программа получает количество монахов и их силы на вход от пользователя. Все эти монахи и их показатели силы записываются в вектор.
Далее начинаются раунды турнира. Вектор с монахами перемешивается. Начинают дуэли между монахами. В новый вектор попадают только монахи победители, с увеличенным
значением силы на силу побеждённого ими монаха, а также монах, которому не достался соперник, с увдоенным значением силы, если число оставшихся монахов не кратно двум.
Данные действия повторяются. Пока не останется один монах, он и будет считаться победителем турнира.

### (6 баллов) Реализован ввод данных из командной строки
Реализован.

### (7 баллов) В программу добавлены ввод данных из файла и вывод результатов в файл.
Добавлен. 

### (8 баллов) В программу добавлена генерация случайных данных в допустимых диапазонах.
Добавлена. 

### Примеры работы программы представлены в папке `test`
