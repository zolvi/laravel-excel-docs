# Row Validation

Sometimes you might want to validate each row before it's inserted into the database. 
By implementing the `WithValidation` concern, you can indicate the rules that each row need to adhere to.

The `rules()` method, expects an array with Laravel Validation rules to be returned. 

```php
<?php

namespace App\Imports;

use App\User;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Concerns\Importable;
use Maatwebsite\Excel\Concerns\WithValidation;

class UsersImport implements ToModel, WithValidation
{
    use Importable;

    public function model(array $row)
    {
        return new User([
            'name'     => $row[0],
            'email'    => $row[1],
            'password' => 'secret',
        ]);
    }

    public function rules(): array
    {
        return [
            '1' => Rule::in(['patrick@maatwebsite.nl']),

             // Above is alias for as it always validates in batches
             '*.1' => Rule::in(['patrick@maatwebsite.nl']),
        ];
    }
}
```

### Validating with a heading row

When using the `WithHeadingRow` concern, you can use the heading row name as rule attribute.

```php
<?php

namespace App\Imports;

use App\User;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Concerns\Importable;
use Maatwebsite\Excel\Concerns\WithValidation;
use Maatwebsite\Excel\Concerns\WithHeadingRow;

class UsersImport implements ToModel, WithValidation, WithHeadingRow
{
    use Importable;

    public function model(array $row)
    {
        return new User([
            'name'     => $row['name'],
            'email'    => $row['email'],
            'password' => 'secret',
        ]);
    }

    public function rules(): array
    {
        return [
            'email' => Rule::in(['patrick@maatwebsite.nl']),

             // Above is alias for as it always validates in batches
             '*.email' => Rule::in(['patrick@maatwebsite.nl']),
        ];
    }
}
```

### Custom validation messages

By adding `customValidationMessages()` method to your import, you can specify custom messages for each failure.


```php
/**
* @return array
*/
public function rules(): array
{
    return [
        '1' => Rule::in(['patrick@maatwebsite.nl']),
    ];
}

/**
 * @return array
 */
public function customValidationMessages()
{
    return [
        '1.in' => 'Custom message for :attribute.',
    ];
}
```

### Custom validation attributes

By adding `customValidationAttributes()` method to your import, you can specify custom attribute names for each column.

```php
/**
* @return array
*/
public function rules(): array
{
    return [
        '1' => Rule::in(['patrick@maatwebsite.nl']),
    ];
}

/**
 * @return array
 */
public function customValidationAttributes()
{
    return ['1' => 'email'];
}
```

## Handling validation errors

### Database transactions

The entire import is automatically wrapped in a **database transaction**, that means that *every* error will rollback the entire import. When using chunked reading, only the **current** chunk will be rollbacked.

### Gathering all failures at the end

You can gather all validation failures at the end of the import. You can try-catch the `ValidationException`. On this exception you can get all failures.

Each failure is an instance of `Maatwebsite\Excel\Validators\Failure`. The `Failure` holds information about which row, which column and what the validation errors are for that cell.

```php
try {
    $import->import('import-users.xlsx');
} catch (\Maatwebsite\Excel\Validators\ValidationException $e) {
     $failures = $e->failures();
     
     foreach ($failures as $failure) {
         $failure->row(); // row that went wrong
         $failure->attribute(); // either heading key (if using heading row concern) or column index
         $failure->errors(); // Actual error messages from Laravel validator
     }
}
```

### Skipping failures

Sometimes you might want to skip failures. By using the `SkipsOnFailure` concern, you get control over what happens the moment a validation failure happens. 
When using `SkipsOnFailure` the entire import will **not** be rollbacked when a failure occurs.

```php
<?php

namespace App\Imports;

use App\User;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Validators\Failure;
use Maatwebsite\Excel\Concerns\Importable;
use Maatwebsite\Excel\Concerns\SkipsOnFailure;
use Maatwebsite\Excel\Concerns\WithValidation;

class UsersImport implements ToModel, WithValidation, SkipsOnFailure
{
    use Importable;

    /**
     * @param Failure[] $failures
     */
    public function onFailure(Failure ...$failures)
    {
        // Handle the failures how you'd like.
    }
}
```

### Skipping errors

Sometimes you might want to skip **all** errors, e.g. duplicate database records. By using the `SkipsOnError` concern, you get control over what happens the moment a model import fails. 
When using `SkipsOnError` the entire import will **not** be rollbacked when an database exception occurs.

```php
<?php

namespace App\Imports;

use App\User;
use Maatwebsite\Excel\Concerns\ToModel;
use Maatwebsite\Excel\Concerns\Importable;
use Maatwebsite\Excel\Concerns\SkipsOnError;
use Maatwebsite\Excel\Concerns\WithValidation;

class UsersImport implements ToModel, WithValidation, SkipsOnError
{
    use Importable;

    /**
     * @param \Throwable $e
     */
    public function onError(\Throwable $e)
    {
        // Handle the exception how you'd like.
    }
}
```

### Row Validation without ToModel

If you are not using the `ToModel` concern, you can very easily do row validation by just using the Laravel validator.

```php
<?php

namespace App\Imports;

use App\User;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Validator;
use Maatwebsite\Excel\Concerns\ToCollection;

class UsersImport implements ToCollection
{
    public function collection(Collection $rows)
    {
         Validator::make($rows->toArray(), [
             '*.0' => 'required',
         ])->validate();

        foreach ($rows as $row) {
            User::create([
                'name' => $row[0],
            ]);
        }
    }
}
```

:::warning Validation rules
For a list of all validation rules, please refer to the [Laravel document](https://laravel.com/docs/validation#available-validation-rules).
:::
