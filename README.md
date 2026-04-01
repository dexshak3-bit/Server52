# Server52
хранимая
-- 21 Создать хранимую процедуру для удаления детали из таблицы P, но только при условии, что на неё нет ссылок в таблице поставок SPJ Если есть ссылки есть - отменить удаление и дывать предупреждение
use supplies;
DELIMITER //
CREATE PROCEDURE Deletes(
    IN p_part_id CHAR(6)
)
BEGIN
    DECLARE reference_count INT DEFAULT 0;
    SELECT COUNT(*) INTO reference_count
    FROM SPJ
    WHERE part_id = p_part_id;
    IF reference_count = 0 THEN
        DELETE FROM P WHERE part_id = p_part_id;
        IF ROW_COUNT() > 0 THEN
            SELECT CONCAT('деталь ', p_part_id, ' удалена') AS message;
        ELSE
            SELECT CONCAT('деталь ', p_part_id, ' не найдена') AS warning;
        END IF;
    ELSE
        SELECT CONCAT('Невозможно удалить деталь ', p_part_id, '.  (', reference_count, ' ссылок)') AS warning;
    END IF;
END //

DELIMITER ;

				Триггеры
MySQL
1.
Запретить вставку или обновление записи, если Дата_окончания_дела указана и она раньше, чем Дата_начала_дела. Если Дата_окончания_дела не указана (NULL) - не проверять.

CREATE TRIGGER check_case_dates_insert
    BEFORE INSERT ON cases
    FOR EACH ROW
BEGIN
    IF NEW.end_date IS NOT NULL AND NEW.end_date < NEW.start_date THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Дата окончания дела не может быть раньше даты начала';
    END IF;
END$$

CREATE TRIGGER check_case_dates_update
    BEFORE UPDATE ON cases
    FOR EACH ROW
BEGIN
    IF NEW.end_date IS NOT NULL AND NEW.end_date < NEW.start_date THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Дата окончания дела не может быть раньше даты начала';
    END IF;
END$$

2.

При добавлении нового клиента проверять, существует ли в таблице Дела запись с Номером_дела, который указан для клиента.

CREATE TRIGGER check_client_case_insert
    BEFORE INSERT ON clients
    FOR EACH ROW
BEGIN
    DECLARE case_count INT;
    
    SELECT COUNT(*) INTO case_count FROM cases WHERE case_number = NEW.case_number;
    
    IF case_count = 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Дело с указанным номером не существует';
    END IF;
END$$

CREATE TRIGGER check_client_case_update
    BEFORE UPDATE ON clients
    FOR EACH ROW
BEGIN
    DECLARE case_count INT;
    
    SELECT COUNT(*) INTO case_count FROM cases WHERE case_number = NEW.case_number;
    
    IF case_count = 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Дело с указанным номером не существует';
    END IF;
END$$

3. Проверять, что Размер_гонорара >= 0.

CREATE TRIGGER check_fee_insert
    BEFORE INSERT ON clients
    FOR EACH ROW
BEGIN
    IF NEW.fee_amount < 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Размер гонорара не может быть отрицательным';
    END IF;
END$$

CREATE TRIGGER check_fee_update
    BEFORE UPDATE ON clients
    FOR EACH ROW
BEGIN
    IF NEW.fee_amount < 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Размер гонорара не может быть отрицательным';
    END IF;
END$$

Детали

MySQL --1. Получить номера изделий, для которых детали полностью 
поставляет поставщик с указанным номером (параметр - номер 
поставщика (S1)) 
SELECT DISTINCT номер_изделия 
FROM SPJ 
WHERE номер_поставщика = 'S1' 
AND номер_изделия NOT IN ( 
SELECT DISTINCT номер_изделия 
FROM SPJ 
WHERE номер_поставщика != 'S1' 
); --2. Получить общее количество деталей с указанным номером, 
поставляемых некоторым поставщиком (параметры - номер детали 
(P1), номер поставщика (S1)). 
SELECT SUM(количество) AS общее_количество 
FROM SPJ 
WHERE номер_детали = 'P1' AND номер_поставщика = 'S1'; 
--3. Выдать номера изделий, использующих только детали, 
поставляемые некоторым поставщиком (параметр - номер 
поставщика (S1)). 
SELECT DISTINCT номер_изделия 
FROM SPJ 
WHERE номер_изделия NOT IN ( 
SELECT DISTINCT номер_изделия 
FROM SPJ 
WHERE номер_поставщика != 'S1' 
); --4. Получить общее число изделий, для которых поставляет детали 
поставщик с указанным номером (параметр - номер поставщика (S1)). 
SELECT COUNT(DISTINCT номер_изделия) AS количество_изделий 
FROM SPJ 
WHERE номер_поставщика = 'S1'; --5. Выдать номера изделий, детали для которых поставляет каждый 
поставщик, поставляющий какую-либо деталь указанного цвета 
(параметр - цвет детали (красный)). 
SELECT номер_изделия 
FROM SPJ 
WHERE номер_поставщика IN ( 
    SELECT DISTINCT номер_поставщика 
    FROM SPJ 
    WHERE номер_детали IN ( 
        SELECT номер_детали 
        FROM P 
        WHERE цвет = 'Красный' 
    ) 
) 
GROUP BY номер_изделия 
HAVING COUNT(DISTINCT номер_поставщика) = ( 
    SELECT COUNT(DISTINCT номер_поставщика) 
    FROM SPJ 
    WHERE номер_детали IN ( 
        SELECT номер_детали 
        FROM P 
        WHERE цвет = 'Красный' 
    ) 
); 
 
 
--6. Получить полный список деталей для всех изделий, 
изготавливаемых в некотором городе (параметр - название города 
(Лондон)). 
SELECT DISTINCT P.* 
FROM P 
JOIN SPJ ON P.номер_детали = SPJ.номер_детали 
JOIN J ON SPJ.номер_изделия = J.номер_изделия 
WHERE J.город = 'Лондон'; --7. Выдать номера деталей, поставляемых каким-либо поставщиком 
из указанного города (параметр - название города (Лондон)). 
SELECT DISTINCT SPJ.номер_детали 
FROM SPJ 
JOIN S ON SPJ.номер_поставщика = S.номер_поставщика 
WHERE S.город = 'Лондон'; --8. Получить список всех поставок, в которых количество деталей 
находится в некотором диапазоне (параметры - границы диапазона 
(от 300 до 750)) 
SELECT * 
FROM SPJ 
WHERE количество BETWEEN 300 AND 750 
ORDER BY количество; 
--9. Выдать номера и названия деталей, поставляемых для какого
либо изделия из указанного города (параметр - название города 
(Лондон)). 
SELECT DISTINCT P.номер_детали, P.название 
FROM P 
JOIN SPJ ON P.номер_детали = SPJ.номер_детали 
JOIN J ON SPJ.номер_изделия = J.номер_изделия 
WHERE J.город = 'Лондон'; --10. Получить цвета деталей, поставляемых некоторым поставщиком 
(параметр - номер поставщика (S1)). 
SELECT DISTINCT P.цвет 
FROM P 
JOIN SPJ ON P.номер_детали = SPJ.номер_детали 
WHERE SPJ.номер_поставщика = 'S1'; --11. Выдать номера и фамилии поставщиков, поставляющих 
некоторую деталь для какого-либо изделия в количестве, большем 
среднего объема поставок данной детали для этого изделия (параметр - номер детали (P1)). 
SELECT DISTINCT S.номер_поставщика, S.фамилия 
FROM S 
JOIN SPJ SPJ1 ON S.номер_поставщика = SPJ1.номер_поставщика 
WHERE SPJ1.номер_детали = 'P1' 
AND SPJ1.количество > ( 
SELECT AVG(количество) 
FROM SPJ SPJ2 
WHERE SPJ2.номер_детали = 'P1' 
AND SPJ2.номер_изделия = SPJ1.номер_изделия --12. Получить названия изделий, для которых поставляются детали 
некоторым поставщиком (параметр - номер поставщика (S1)). 
SELECT DISTINCT J.название 
FROM J 
JOIN SPJ ON J.номер_изделия = SPJ.номер_изделия 
WHERE SPJ.номер_поставщика = 'S1';

представления
 
MySQL: --1 Представление "Подзащитные, осужденные условно со сроком более 
одного года"  
CREATE OR REPLACE VIEW probation_clients_over_one_year AS 
SELECT  
    c.passport_number, 
    c.full_name, 
    c.birth_date, 
    cs.case_number, 
    cs.start_date, 
    cs.end_date, 
    c.result, 
    c.sentence_term, 
    TIMESTAMPDIFF(YEAR, c.birth_date, CURDATE()) AS current_age, 
    CASE  
        WHEN c.sentence_term > 1 THEN 'срок более 1 года' 
        WHEN c.sentence_term <= 1 THEN 'срок до 1 года' 
    END AS term_category 
FROM clients c 
JOIN cases cs ON c.case_number = cs.case_number 
WHERE c.result = 'осужден условно'  
  AND c.sentence_term > 1 
ORDER BY c.sentence_term DESC, cs.start_date; 
 --2 Представление "Эффективность защиты": дело ФИО (максимальный 
срок - срок поприговору) (срок по приговору - минимальный срок). 
Минимальный и максимальный сроки должны выбираться среди всех 
статей, по которым обвинялся клиент в рамках одного дела.  
CREATE OR REPLACE VIEW defense_effectiveness_mysql AS 
WITH client_article_terms AS ( 
    SELECT  
        c.passport_number, 
        c.full_name, 
        cs.case_number, 
        cs.start_date, 
        cs.end_date, 
        c.result, 
        c.sentence_term, 
        MIN(acc.min_term) AS min_possible_term, 
        MAX(acc.max_term) AS max_possible_term, 
        GROUP_CONCAT(DISTINCT acc.article ORDER BY acc.article 
SEPARATOR ', ') AS articles_list 
    FROM clients c 
    JOIN cases cs ON c.case_number = cs.case_number 
    LEFT JOIN accusations a ON c.passport_number = a.client_passport 
    LEFT JOIN articles_criminal_code acc ON a.article_code = acc.article 
    GROUP BY c.passport_number, c.full_name, cs.case_number, 
cs.start_date,  
             cs.end_date, c.result, c.sentence_term 
) 
SELECT  
    passport_number, 
    full_name, 
    case_number, 
    start_date, 
    end_date, 
    result, 
    sentence_term, 
    min_possible_term, 
    max_possible_term, 
    articles_list, 
    max_possible_term - sentence_term AS reduction_from_max, 
    sentence_term - min_possible_term AS excess_over_min, 
    ROUND(((max_possible_term - sentence_term) / 
NULLIF(max_possible_term, 0) * 100), 1) AS reduction_percent, 
    CASE  
        WHEN result = 'оправдан' THEN 'Полный успех (оправдан)' 
        WHEN result = 'осужден условно' AND sentence_term < 
min_possible_term THEN 'Условно ниже минимума' 
        WHEN sentence_term < min_possible_term THEN 'Ниже 
минимального срока' 
        WHEN sentence_term <= min_possible_term THEN 'На минимальном 
сроке' 
        WHEN sentence_term < max_possible_term THEN 'Ниже 
максимального' 
        WHEN sentence_term = max_possible_term THEN 'На максимальном 
сроке' 
        WHEN sentence_term > max_possible_term THEN 'Превышает 
максимум' 
        ELSE 'Не определено' 
    END AS effectiveness_rating 
FROM client_article_terms 
ORDER BY  
    CASE WHEN reduction_from_max IS NULL THEN 1 ELSE 0 END, 
    reduction_from_max DESC; 
 --3 Представление "Список статей": номер дела номер статьи УК.  
 CREATE OR REPLACE VIEW case_articles_list AS 
SELECT DISTINCT 
    cs.case_number, 
    a.article_code AS article_number, 
    acc.min_term, 
    acc.max_term, 
    c.full_name AS client_name, 
    c.passport_number 
FROM cases cs 
JOIN clients c ON cs.case_number = c.case_number 
JOIN accusations a ON c.passport_number = a.client_passport 
JOIN articles_criminal_code acc ON a.article_code = acc.article 
ORDER BY cs.case_number, a.article_code;

Настроим проект
В Program.cs необходимо зарегистрировать контекст бд:
builder.Services.AddDbContext<TradeContext>();
Регистрируем класс TradeContext как сервис с областью действия Scoped.
Scoped: Один экземпляр создаётся для каждого HTTP-запроса и используется в течение его обработки.
Также поступим с другими классами, используемыми в HTTP-запросах:
using Sieve.Services;
using TradeProject.Classes;
using TradeProject.Model;
var builder = WebApplication.CreateBuilder(args);
// Регистрация сервисов
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<ValidationHelper>();
builder.Services.AddScoped<SieveProcessor>();
builder.Services.AddDbContext<TradeContext>();
var app = builder.Build();
// Конфигурация middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();

Узнать и изменить порт, по которому можно обратиться к API можно в файле launchSettings.json. (Properties – launchSettings.json)

Контроллеры
Контроллеры в ASP.NET Core используются для обработки HTTP-запросов и управления логикой приложения. Они принимают входящие запросы от клиента, выполняют нужные действия и возвращают результаты в виде ответов.
Теперь создадим такой для product: новый файл ProductController.cs в папке Controllers.

 Убираем метод Index. 
После namespace добавляем атрибуты:
[Route("api/[controller]")]: устанавливает базовый маршрут для контроллера, автоматически заменяя [controller] на имя контроллера без суффикса Controller (например, api/product для ProductController).
[ApiController]: обозначает класс как API-контроллер, включая автоматическую валидацию модели и упрощенную маршрутизацию, что улучшает обработку запросов и ошибок.
 Пропишем инъекцию зависимостей.

hamespace TradeProject.Controllers
{
[Route("api/[controller]")]
[ApiController]
Ссылок: 1
public class ProductController: Controller
{
private readonly ValidationHelper _validation;
private readonly TradeContext _context;
private readonly SieveProcessor sieveProcessor;
Ссылок: 0
public ProductController(TradeContext context, Validation Helper validation Helper, SieveProcessor sieveProcessor)
{
_validation validationHelper;
_context context;
_sieveProcessor = sieveProcessor;

HTTP-запросы
Вот краткое описание основных атрибутов для работы с HTTP-запросами в ASP.NET Core:
⦁	[HttpGet]: обрабатывает HTTP GET-запросы (получение данных).
⦁	[HttpPost]: обрабатывает HTTP POST-запросы (создание нового ресурса).
⦁	[HttpPut]: обрабатывает HTTP PUT-запросы (обновление существующего ресурса).
⦁	[HttpDelete]: обрабатывает HTTP DELETE-запросы (удаление ресурса).
⦁	[HttpPatch]: обрабатывает HTTP PATCH-запросы (частичное обновление ресурса).
⦁	[HttpHead]: обрабатывает HTTP HEAD-запросы (получение заголовков без тела ответа).
⦁	[HttpOptions]: обрабатывает HTTP OPTIONS-запросы (получение поддерживаемых методов для ресурса).
⦁	[HttpPut("{id}")]: используется для указания маршрута с параметрами (например, обновление ресурса с заданным id).
⦁	[Route("path")]: устанавливает маршрут для контроллера или метода.
⦁	[Authorize]: ограничивает доступ к методу или контроллеру только авторизованным пользователям.
⦁	[AllowAnonymous]: разрешает доступ к методу или контроллеру для анонимных пользователей.
⦁	[FromBody]: указывает, что параметр метода должен быть привязан из тела запроса.
⦁	[FromQuery]: указывает, что параметр метода должен быть привязан из строки запроса (параметры URL).
⦁	[FromRoute]: указывает, что параметр метода должен быть привязан из маршрута URL.
⦁	[FromHeader]: указывает, что параметр метода должен быть привязан из заголовков запроса.
⦁	[FromForm]: указывает, что параметр метода должен быть привязан из формы запроса.
⦁	[FromServices]: внедряет зависимость в метод контроллера.
Получив классы в папке Model из базы данных, на их основе, настоятельно рекомендуется, создать классы DTO для обмена данными в HTTP-запросах.
DTO (Data Transfer Object) помогают упростить работу с API, обеспечивая:
⦁	Изоляцию слоев: защита внутренней логики приложения.
⦁	Оптимизацию данных: передача только нужных данных.
⦁	Гибкость: поддержка изменений без затруднений.
Также помогает избежать проблемы с рекурсией (самоссылками), при передаче данных, с внешними ссылками, ссылающихся друг на друга.
Каждый метод, позволяющий связаться с ним, имеет атрибут [Http*действие*]
Task<IActionResult>: Асинхронный метод, который возвращает объект типа IActionResult. Это позволяет возвращать различные результаты (например, Ok(), NotFound(), BadRequest()).
HttpGet
[HttpGet]
public async Task<IActionResult> GetProducts([FromQuery] SieveModel sieveModel)
{
}
.Include(p => p.ProductType)
.Include(p => p.Supplier).AsQueryable();
var products = await sieveProcessor. Apply(sieveModel, query).ToListAsync();
var settings = new JsonSerializerSettings
{
};
PreserveReferences Handling = PreserveReferences Handling.Objects, Formatting = Formatting.Indented
string json = JsonConvert.SerializeObject(products, settings); return ok(json);
}

Как видим, здесь метод Get, для получения списка продуктов, поддерживающий фильтрацию и сортировку, требующий SieveModel в качестве параметра, а перед ним атрибут [FromQuery], который указывает, что параметр метода должен быть привязан из строки запроса (параметры URL). К примеру, запустив проект, в адресной строке можно перейти по адресу:
https://localhost:44330/api/Product?Filters=ManufacturerId==2
Мы получим отформатированную JSON строку:

{
"$id": "1",
"ProductArticleNumber": "8025Y5",
"Name": "Блюдо",
"Measure": "wr.",
"Cost": 2000.0000,
"Description": "блдо декоративное flower 35 Cм Home Philosophy",
"ProductTypeId": 8,
"Photo":"",
"SupplierId": 1,
"ProductMaxDiscount": 3,
"ManufacturerId": 2,
"CurrentDiscount": 3,
"Status":"",
"QuantityInStock": 2, "Manufacturer": { "$id": "2",
"ManufacturerId": 2,
"Name": "Home Philosophy", "Products": [
{
"$ref": "1"
},
