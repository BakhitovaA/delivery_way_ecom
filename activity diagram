```
@startuml
title Алгоритм обработки заявки на создание доставки

start

:Получить событие DELIVERY_CREATION_REQUESTED;

:Проверить данные заявки;

if (Данные корректны?) then (Да)
    :Отправить запрос партнёру\nPOST /deliveries;

    if (Партнёр подтвердил создание?) then (Да)
        :Сохранить partnerDeliveryId;
        :Опубликовать DELIVERY_CREATED;
        stop
    else (Нет)
        :Определить тип ошибки;

        if (Ошибка временная?) then (Да)
            :Выполнить повторную попытку;
            if (Retry успешен?) then (Да)
                :Сохранить partnerDeliveryId;
                :Опубликовать DELIVERY_CREATED;
                stop
            else (Нет)
                :Опубликовать DELIVERY_CREATION_FAILED;
                stop
            endif
        else (Нет)
            :Опубликовать DELIVERY_CREATION_FAILED;
            stop
        endif
    endif

else (Нет)
    :Зафиксировать причину ошибки;
    :Опубликовать DELIVERY_CREATION_FAILED;
    stop
endif

@enduml
```
