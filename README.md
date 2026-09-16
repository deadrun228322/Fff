#include <algorithm>
#include <iomanip>
#include <iostream>
#include <limits>
#include <random>
#include <vector>

using namespace std;

int main() {
    // ---------- Параметры системы (задаются преподавателем/пользователем) ----------
    // Условие устойчивости: lambda < SERVERS * mu
    const double lambda = 0.5;      // интенсивность входного потока
    const double mu     = 0.4;      // интенсивность обслуживания одним сервером

    const int SERVERS = 2;          // число серверов (каналов)
    const int N       = 30;         // задач в одном прогоне
    const int RUNS    = 100000;     // число прогонов (статистических испытаний)

    // ---------- Интервалы для функции распределения: 0-1, 1-2, 2-3, ... ----------
    const int    BINS = 50;         // число интервалов
    const double BW   = 1.0;        // ширина интервала = 1
    // hist[i] — сколько задач попало в интервал [i, i+1) времени ожидания
    vector<long long> hist(BINS, 0);
    long long totalCount = 0;       // всего обслуженных задач (для F(t))
    double waitSum = 0.0;           // сумма времён ожидания

    // ---------- ГПСЧ ----------
    mt19937 mt(42);
    exponential_distribution<> arrival(lambda);  // интервалы между приходами ~ Exp(lambda)
    exponential_distribution<> service(mu);      // время обслуживания       ~ Exp(mu)

    const double INF = numeric_limits<double>::infinity();

    // ---------- Основной цикл: RUNS независимых прогонов ----------
    for (int run = 0; run < RUNS; ++run) {
        int busy = 0;                                // число занятых серверов
        vector<double> queueArrival;                 // времена прихода задач в очереди
        vector<double> busyUntil(SERVERS, INF);      // когда освободится каждый сервер

        int inCount  = 0;
        int outCount = 0;
        double T1 = arrival(mt);                     // время следующего IN

        while (inCount < N || outCount < N) {
            // Ближайшее событие OUT: минимальное время освобождения сервера
            double T2 = INF;
            int freeIdx = -1;
            for (int s = 0; s < SERVERS; ++s)
                if (busyUntil[s] < T2) { T2 = busyUntil[s]; freeIdx = s; }

            if (T1 <= T2) {
                // -------- Событие IN: пришла новая задача --------
                double systemtime = T1;
                ++inCount;

                if (busy < SERVERS) {
                    // Есть свободный сервер — обслуживание начинается сразу, ожидание = 0
                    int s = -1;
                    for (int i = 0; i < SERVERS; ++i)
                        if (busyUntil[i] == INF) { s = i; break; }

                    ++busy;
                    busyUntil[s] = systemtime + service(mt);

                    ++totalCount;
                    waitSum += 0.0;
                    hist[0]++;                       // ожидание 0 попадает в [0,1)
                } else {
                    // Все серверы заняты — задача встаёт в очередь
                    queueArrival.push_back(systemtime);
                }

                T1 = (inCount < N) ? systemtime + arrival(mt) : INF;
            } else {
                // -------- Событие OUT: сервер freeIdx освободился --------
                double systemtime = T2;
                --busy;
                busyUntil[freeIdx] = INF;
                ++outCount;

                if (!queueArrival.empty()) {
                    // Берём СЛУЧАЙНУЮ задачу из очереди
                    uniform_int_distribution<> pick(0, (int)queueArrival.size() - 1);
                    int idx = pick(mt);
                    double arr = queueArrival[idx];
                    queueArrival[idx] = queueArrival.back();
                    queueArrival.pop_back();

                    double wait = systemtime - arr;  // время ожидания в очереди
                    waitSum += wait;
                    ++totalCount;

                    int b = (int)(wait / BW);
                    if (b < 0) b = 0;
                    if (b >= BINS) b = BINS - 1;     // хвост — в последний интервал
                    hist[b]++;

                    // Занимаем освободившийся сервер
                    ++busy;
                    busyUntil[freeIdx] = systemtime + service(mt);
                }
            }
        }
    }

    // ---------- Вывод: эмпирическая функция распределения F(t) ----------
    cout << fixed << setprecision(4);
    cout << "Параметры: lambda = " << lambda
         << ", mu = " << mu
         << ", серверов = " << SERVERS
         << ", прогонов = " << RUNS
         << ", задач в прогоне = " << N << "\n\n";

    cout << "Среднее время ожидания в очереди: "
         << (totalCount ? waitSum / totalCount : 0.0) << "\n\n";

    cout << "Эмпирическая функция распределения времени ожидания F(t):\n";
    cout << setw(12) << "интервал"
         << setw(18) << "F(t)" << "\n";

    long long acc = 0;
    for (int i = 0; i < BINS; ++i) {
        acc += hist[i];
        double F = (double)acc / totalCount;   // накопленная частота
        cout << setw(6) << i << " - " << setw(4) << (i + 1)
             << setw(18) << F << "\n";
    }

    return 0;
}
