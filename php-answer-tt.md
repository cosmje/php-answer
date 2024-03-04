# Исходный код:
```php
<?php

namespace vBulletin\Search;

class Search {
    public static function doSearch(): void {
      if ($_REQUEST['searchid']){
        $_REQUEST['do'] = 'showresults';
      }elseif(!empty($_REQUEST['q'])){
          $_REQUEST['do'] = 'process';
          $_REQUEST['query'] = &$_REQUEST['q'];
      }
      $db = new \PDO('mysql:dbname=vbforum;host=127.0.0.1', 'forum', '123456');
      if ($_REQUEST['do'] == 'process') {
          $sth = $db->prepare('SELECT * FROM vb_post WHERE text like ?');
          $sth->execute(array($_REQUEST['query']));
          $result = $sth->fetchAll();
          self::render_search_results($result);
          $file = fopen('/var/www/search_log.txt', 'a+');
          fwrite($file, $_REQUEST['query'] . "\n");
      } elseif ($_REQUEST['do'] == 'showresults'){
          $sth = $db->prepare('SELECT * FROM vb_searchresult WHERE searchid = ?');
          $sth->execute(array($_REQUEST['searchid']));
          $result = $sth->fetchAll();
          self::render_search_results($result);
      }
      else {
          echo "<h2>Search in forum</h2><form><input name='q'></form>";
      }
  }

  public static function render_search_results($result){
      global $render;
      foreach($result as $row){
          if ($row['forumid'] != 5){
              $render->render_searh_result($row);
          }
      }
  }
}
```

# Вопрос 1
## "1. Напишите, какие есть проблемы у этого кода?"

# Ответ 1.

> [!TIP]
>### - Данные для подключения к БД (имя пользователя, пароль, имя базы данных и хост) захардкодены внутри класса;
>### - Небезопасное использование инпутов из поисковой формы;
>### - Бизнес-логика и презентация перемешаны и использованы одновременно внутри класса; 
>### - Записывает логи при помощи fwrite() вместо использования предназначенной для этого функции функции error_log();
>### - Код не содержит обработчика ошибок для запросов к БД, что может привести к непредсказуемому поведению в случае возникновения проблем;
>### - Функция render_search_results() использует глобальную переменную $render, что делает код менее чистым. Лучше передавать $render в качестве аргумента функции;
>### - Были использованы сложночитаемые названия таблиц и переменных: showresults, searchid, render_search_results, vb_searchresult и тд;
>### - Отсутствие комментариев. 


# Вопрос 2
## "2. Сделайте рефакторинг данного кода. Можно изменять названия методов, классов,
 добавлять входящие параметры, создавать новые классы и т.п.. Можно использовать
 псевдокод."

# Ответ 2.

## 2.1 - Создаем .env файл: 
```
DB_HOST=127.0.0.1
DB_NAME=vbforum
DB_USERNAME=forum
DB_PASSWORD=123456
```

## 2.2 - Создаем config.php файл: 
```php
<?php

$dotenv = Dotenv\Dotenv::createImmutable(__DIR__);
$dotenv->load();

$DB_HOST = $_ENV['DB_HOST'];
$DB_NAME = $_ENV['DB_NAME'];
$DB_USERNAME = $_ENV['DB_USERNAME'];
$DB_PASSWORD = $_ENV['DB_PASSWORD'];
?>
```


## 2.2 - Создаем search_form.php файл во view: 
```html
<h2>Поиск по форуму</h2>
<form method="GET" action="Search.php">
    <label for="q">Совершить поиск по ключевым словам:</label>
    <input type="text" name="q">
    <button type="submit">Искать</button>
</form>
```

## 2.3 - Редактируем исходный код:

```php
<?php

namespace vBulletin\Search;

use PDO;
use Exception;
use vBulletin\Render;

require_once 'config.php';

class Search
{
    private $db;
    private $render;
    private $hiddenForumIds;

    public function __construct(PDO $db, Render $render)
    {
        $this->db = $db;
        $this->render = $render;
        $this->hiddenForumIds = $this->fetchHiddenForumIds();
    }



    // ПОЛУЧАЕМ ТАБЛИЦУ СО СКРЫТЫМИ ФОРУМАМИ
    private function fetchHiddenForumIds()
    {
        // все forum_id, которые не должны выдаваться во время поиска,
        // записаны в отдельную таблицу hidden_forum_ids
        // а затем в переменную $hiddenForumIds

        $query = "SELECT forum_id FROM hidden_forum_ids";
        $statement = $this->db->query($query);
        $hiddenForumIds = $statement->fetchAll(PDO::FETCH_COLUMN);
        return $hiddenForumIds;
    }



    // УСТАНАВЛИВАЕМ СОЕДИНЕНИЕ С БД
    private function connectDB()
    {
        try {

            // получаем данные входа из config.php
            // сам config.php ссылается на .env файл

            $db = new PDO("mysql:host=" . $DB_HOST . ";dbname=" . $DB_NAME, $DB_USERNAME, $DB_PASSWORD); 
            $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION); // для мониторинга ошибок
            return $db;
        } catch (PDOException $e) {
            throw new Exception("Ошибка подключения к БД: " . $e->getMessage());
        }
    }



    // РАСПРЕДЕЛЕНИЕ ПОИСКОВОГО ЗАПРОСА ПО МЕТОДАМ
    public function doSearch(): void
    {
        // в зависимости от того, что содержится в запросе, 
        // будет выполняться соответсвующий метод

        $action = $_REQUEST['do'] ?? '';

        switch ($action) {
            case 'process':
                $this->processSearch();
                break;
            case 'show_results':
                $this->showSearchResults();
                break;
            default:
                $this->renderSearchForm();
        }
    }



    // ОБРАБОТКА В СЛУЧАЕ ИНПУТА ПОЛЬЗОВАТЕЛЯ (ТЕКСТ)
    private function processSearch()
    {
        $query = $_REQUEST['q'] ?? '';

        if (empty($query)) {
            $this->renderErrorMessage("Запрос пуст.");
            return;
        }

        $db = $this->connectDB();

        try {
            $sth = $db->prepare('SELECT * FROM vb_post WHERE text LIKE :query');

            // переменная :query будет заменена на фактическое значение при выполнении запроса, 
            // И добавлена, чтобы повысить защиту от SQL-иньекций

            $sth->bindValue(':query', "%$query%", PDO::PARAM_STR); // значение будет строковым
            $sth->execute();
            $result = $sth->fetchAll();
            $this->renderSearchResults($result);
            $this->logSearchQuery($query);
        } catch (PDOException $e) {
            $this->renderErrorMessage("Ошибка базы данных: " . $e->getMessage());
        }
    }



    // ОТОБРАЖЕНИЕ РЕЗУЛЬТАТОВ ПОИСКА ПО SEARCH_ID 
    private function showSearchResults()
    {
        $searchId = isset($_REQUEST['search_id']) ? (int)$_REQUEST['search_id'] : 0;

        if ($searchId <= 0) {
            $this->renderErrorMessage("Недопустимый идентификатор поиска.");
            return;
        }

        $db = $this->connectDB();

        try {
            $sth = $db->prepare('SELECT * FROM vb_search_result WHERE search_id = :search_id');
            $sth->bindValue(':search_id', $searchId, PDO::PARAM_INT); // значение будет числовым
            $sth->execute();
            $result = $sth->fetchAll();
            $this->renderSearchResults($result);
        } catch (PDOException $e) {
            $this->renderErrorMessage("Ошибка базы данных: " . $e->getMessage());
        }
    }



    // ФОРМА ПОИСКА 
    private function renderSearchForm()
    {
        // получаем форму поиска из вьюшки
        include 'search_form.php';
    }



    // ОБРАБОТКА ПОИСКОВОГО ЗАПРОСА
    private function renderSearchResults($result)
    {
        foreach ($result as $row) {

            // если forum_id окажется в списке hiddenForumIds, 
            // то отображаться в результате поиска он не будет

            if (!in_array($row['forum_id'], $this->hiddenForumIds)) {
                $this->render->renderSearchResult($row);
            }
        }
    }



    // ЗАПИСЬ ЗАПРОСОВ В ЛОГ 
    private function logSearchQuery($query)
    {
        // предварительно обрабатываем инпут пользователя перед сохранением записи в лог 
        $escapedQuery = htmlspecialchars($query, ENT_QUOTES, 'UTF-8');
        
        //записываем логи при помощи специально предназначенной для этого функции error_log()
        error_log('Поисковый запрос: ' . $escapedQuery, 3, '/var/www/search_log.txt');
    }



    // СООБЩЕНИЯ ОБ ОШИБКАХ (если есть)
    private function renderErrorMessage($message)
    {
        echo "<p>Ошибка: $message</p>";
    }
}

```
