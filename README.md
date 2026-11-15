#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include<string.h>
#include<time.h>
#define MAX_LOGS 100
time_t maketime(int year, int month, int day, int hour, int minute, int second)
{
    struct tm value = { 0 };
    value.tm_year = year - 1900;
    value.tm_mon = month - 1;
    value.tm_mday = day;
    value.tm_hour = hour;
    value.tm_min = minute;
    value.tm_sec = second;
    value.tm_isdst = -1;
    time_t result= mktime(&value);
    if (result == (time_t)-1||
        value.tm_year != year - 1900 ||
        value.tm_mon != month - 1 ||
        value.tm_mday != day ||
        value.tm_hour != hour ||
        value.tm_min != minute ||
        value.tm_sec != second)
    {
        return (time_t)-1;
    }
    return result;
}
// 将当天时间换算为从零点开始的秒数
int secondsf(int hour, int minute, int second)
{
    return hour * 3600 + minute * 60 + second;
}
// 保存一条登录日志及其检查结果
struct Log
{
    char date[11];
    char time[9];
    char ip[16];
    char user[32];
    char result[16];
    int valid;
    int seconds;
    time_t timestamp;
};
int main(void)
{
    char line[256];
    struct Log record[MAX_LOGS] = { 0 };
    int logCount = 0;
    char targetIP[64];
    int failcount = 0;
    // 输入要单独查询的 IP 地址
    printf("Enter an IP to check");
    if (fgets(targetIP, sizeof targetIP, stdin) == NULL)
    {
        printf("Input error\n");
        return 1;
    }
    targetIP[strcspn(targetIP, "\n")] = '\0';
    // 打开工作目录中的日志文件
    FILE* file = fopen("logs.txt", "r");
    if (file == NULL)
    {
        printf("Cannot open logs.txt\n");
        return 1;
    }
    char result1[8] = "SUCCESS", result2[5] = "FAIL";
    int s = 0, f = 0, u = 0;
    // 逐行读取日志，拆分字段并检查时间
    while(logCount<MAX_LOGS)
    {
        int i = logCount;
        if (fgets(line, sizeof line, file) == NULL)
        {
            break;
        }
        int count = sscanf(line, "%10s %8s %15s %31s %15s",
            record[i].date,
            record[i].time,
            record[i].ip, 
            record[i].user, 
            record[i].result);
        // valid 表示通过当前已有的字段与时间检查
        record[i].valid = (count == 5);
        if (count == 5)
        {
            int hour, minute, second;
            if (sscanf(record[i].time, "%d:%d:%d", &hour, &minute, &second) == 3 &&
                hour >= 0 && hour < 24 &&
                minute >= 0 && minute < 60 &&
                second >= 0 && second < 60)
            {
                record[i].seconds = secondsf(hour, minute, second);
                int year, month, day;
                if (sscanf(record[i].date, "%d-%d-%d", &year, &month, &day) == 3)
                {
                    record[i].timestamp = maketime(year, month, day, hour, minute, second);
                    if (record[i].timestamp == (time_t)-1)
                    {
                        record[i].valid = 0;
                        printf("Time conversion failed at log %d\n",i+1);
                    }
                }
                else
                {
                    record[i].valid = 0;
                    printf("Invalid date at log %d\n", i + 1);
                }
                printf("Time: %s -> %d\n", record[i].time, record[i].seconds);
            }
            else
            {
                record[i].valid = 0;
                printf("Invalid time at log %d\n", i + 1);
            }
            if (record[i].valid == 1)
            {
                if (strcmp(result1, record[i].result) == 0)
                {
                    s++;
                }
                else if (strcmp(result2, record[i].result) == 0)
                {
                    f++;
                }
                else
                {
                    u++;
                }
            }
        }
        else
            printf("Invalid log line\n");
        logCount++;
    }
    if (logCount == MAX_LOGS)
    {
        printf("Reached the limit of %d log lines.\n", MAX_LOGS);
    }
    fclose(file);
    // 依次以每条有效的失败记录作为时间窗口的起点
    for (int i = 0; i < logCount; i++)
    {
        if (record[i].valid == 0 ||
            strcmp(record[i].result, "FAIL") != 0)
        {
            continue;
        }

        int windowcount = 0;
        // 遍历所有记录，查找同一 IP 在窗口内的失败次数
        for (int j = 0; j < logCount; j++)
        {
            if (record[j].valid == 1 &&
                strcmp(record[j].result, "FAIL") == 0 &&
                strcmp(record[i].ip, record[j].ip) == 0)
            {
                double difference = difftime(record[j].timestamp,record[i].timestamp);
                // 只统计起点之后 300 秒内的记录，包含起点和边界
                if (difference >= 0 && difference <= 300)
                {
                    windowcount++;
                }
            }
        }
        if (windowcount >= 5)
        {
            printf("Warning: %s had %d failures within 5 minutes from %s %s\n",
                record[i].ip,
                windowcount,
                record[i].date,
                record[i].time);
       }
    }// 演示：比较同一天前两条记录的时间差
    for (int i = 0;i < logCount;i++)
    {
        if (record[i].valid == 0)
        {
            continue;
        }
        // 检查当前 IP 是否在之前的有效记录中出现过
        int seen = 0;
        for (int j = 0;j < i; j++)
        {
            if (record[j].valid == 1 && strcmp(record[i].ip, record[j].ip) == 0)
            {
                seen = 1;
                break;
            }
        }
        if (seen == 0)
        {
            int ipFailCount = 0;
            for (int j = 0;j < logCount;j++)
            {
                if (record[j].valid == 1 &&
                    strcmp(record[i].ip, record[j].ip) == 0 &&
                    strcmp(record[j].result, "FAIL") == 0)
                {
                    ipFailCount++;
                }
            }
            printf("%s: %d failed logins\n", record[i].ip, ipFailCount);
        }// 统计这个 IP 在所有已读取记录中的失败次数
    }
    for (int i = 0;i < logCount;i++)
    {
        if (record[i].valid == 1 &&
            strcmp(record[i].ip, targetIP) == 0 &&
            strcmp(record[i].result, "FAIL") == 0)
        {
            failcount++;
        }
    }// 单独统计用户输入的 IP 的失败次数
    printf("Failed logins from %s: %d\n", targetIP, failcount);
    return 0;
}
