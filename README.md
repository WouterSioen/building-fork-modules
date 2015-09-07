# How I build Fork CMS modules

---

## Hi, I'm Wouter

![Sumo Wouter](img/Sumo_Wouter.png)

:twitter: [@WouterSioen](http://twitter.com/WouterSioen)

:github: [WouterSioen](http://github.com/WouterSioen)

---

## I work at Sumocoders

---

![I mainly use Symfony](img/symfony.png)

---

# I'm a Fork core developer

---

# How I build Fork CMS modules

---

<https://github.com/woutersioen/moduleMaker>

---

## Module Maker

1. Click "Download zip" on github.
2. Upload in the Fork CMS backend.
3. Click install.

---

## Custom module: a Team module

---

## 1. Dependencies

---

```bash
composer require ramsey/uuid
```

---

### UUID's

* Independent of your Database
* Available before insert
* Hides internals

---

## 2. Entity

???

We use an object to represent our object. This is the easiest to know our
team member is always in a valid state. It makes it easy to add validation in
once place instead of scattered all around your code.

It also allows you to typehint on it. When you typehint "array", you can expect
everything to be send to the method (and have to handle all cases). When you
typehint for an Entity, you know what to expect. All possibilities are written
in it's public properties/methods

---

```php
<?php

namespace Backend\Modules\Team\Entity;

final class TeamMember
{
    private $id;
    private $metaId;
    private $language;
    private $name;
    private $description;
    private $isHidden = false;
    private $editedOn;
    private $createdOn;
}
```

???

The class is final to make sure it will not be extend. I also make my
properties private.

This way, I'm sure my entity can't be altered in ways I don't intended using
my public methods.

---

```php
private function __construct(
    $id,
    $metaId,
    $language,
    $name,
    $description,
    $isHidden,
    \DateTime $editedOn,
    \DateTime $createdOn
) {
    $this->id = $id;
    $this->metaId = $metaId;
    $this->language = $language;
    $this->name = $name;
    $this->description = $description;
    $this->isHidden = $isHidden;
    $this->editedOn = $editedOn;
    $this->createdOn = $createdOn;
}
```

???

My constructor contains ALL properties that are required in the database. This
way, I can't create an entity that has missing properties, and throws database
exceptions.

---

```php
public static function create(
    $metaId,
    $language,
    $name,
    $description
) {
    return new self(
        Uuid::uuid4(),
        $metaId,
        $language,
        $name,
        $description,
        false,
        new \DateTime(),
        new \DateTime()
    );
}
```

???

I made my constructor private, because I want an easier api to create my
TeamMember objects. Now I've added this create method.

We call this a named constructor. It only receives 4 parameters, and does the
job of adding the other 4, providing us with less work in all other places that
add TeamMembers.

---

```php
public function change($metaId, $name, $description)
{
    $this->metaId = $metaId;
    $this->name = $name;
    $this->description = $description;
    $this->editedOn = new \DateTime();
}
```

???

I avoid to use setters on Entities (and objects in general). When you see
entities "in the wild", you mostly see a constructor without arguments and a lot
of setters. This again allows us to create invalid Entities, that will give
errors when persisting to the database.

A nice alternative to setters is methods that change internal variables in one
group. In this case, it's just an edit action, but a nice example could be a
Person with a "relocate" method, instead of "setStreet", "setNumber", "setCity",...

---

```php
public static function fromArray(array $teamMemberArray)
{
    return new self(
        Uuid::fromBytes($teamMemberArray['id']),
        $teamMemberArray['meta_id'],
        $teamMemberArray['language'],
        $teamMemberArray['name'],
        $teamMemberArray['description'],
        (bool) $teamMemberArray['is_hidden'],
        \DateTime::createFromFormat(
            'Y-m-d H:i:s',
            $teamMemberArray['edited_on']
        ),
        \DateTime::createFromFormat(
            'Y-m-d H:i:s',
            $teamMemberArray['created_on']
        )
    );
}
```

???

We also create a second named constructor. It creates a teamMember from an array.

This will be used when we fetched our teamMember array from the database.

---

```php
public function getId()
{
    return $this->id;
}

public function getMetaId()
{
    return $this->metaId;
}

public function getName()
{
    return $this->name;
}

public function getDescription()
{
    return $this->description;
}
```

???

Our entity off course needs some getters. We need to fetch some data in serveral
places. I only add getters for stuff thats used. No need for dead code.

---

```php
public function toArray()
{
    return [
        'id' => $this->id,
        'meta_id' => $this->metaId,
        'language' => $this->language,
        'name' => $this->name,
        'description' => $this->description,
        'is_hidden' => $this->isHidden,
        'edited_on' => $this->editedOn,
        'created_on' => $this->createdOn,
    ];
}
```

???

Last part of the Entity is the toArray method. When using an ORM, you won't need
this one (and the fromArray method), but it's useful in Fork to convert it for
our templates and our database system.

---

## 2. Installer

---

```sql
CREATE TABLE IF NOT EXISTS `team_members` (
  `id`          binary(16)    NOT NULL COMMENT 'a binary representation of a UUID',
  `meta_id`     int(11)       NOT NULL,
  `language`    varchar(5)    NOT NULL,
  `name`        varchar(255)  NOT NULL,
  `description` text          NOT NULL,
  `edited_on`   datetime      NOT NULL,
  `created_on`  datetime      NOT NULL,
  `is_hidden`   tinyint(1)    NOT NULL DEFAULT 0 COMMENT 'lets not use ENUMs for this',
  PRIMARY KEY  (`id`),
  KEY `hidden` (`hidden`),
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci ;
```

???

Some people store uuid's as VARCHAR(40) rows. This lowers the database performance
though, especially with large tables. I store it as BINARY(16). This gives us
back our full performance.

I also save booleans as TINYINT(1) instead of ENUM('Y','N'). This is the most
generally used way to save booleans in a MySQL database.

---

> designing a class that can be meaningfully inherited from takes more than just removing a final specifier; it takes a lot of care.
>
> -- Konrad Rudolph

???

My installer.php and locale.xml file are just default. Nothing to see there.
The only special thing to say about it is that I make my Installer classes final.
Why would you ever like to extend an Installer?

My Config file is also final.

---

## 3. Module Extension

---

```php
<?php

namespace Backend\Modules\Team\DependencyInjection;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\Config\FileLocator;
use Symfony\Component\HttpKernel\DependencyInjection\Extension;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

final class TeamExtension extends Extension
{
    public function load(array $configs, ContainerBuilder $container)
    {
        $loader = new YamlFileLoader(
            $container,
            new FileLocator(__DIR__.'/../Resources/config')
        );
        $loader->load('services.yml');
    }
}
```

???

Since some versions, Fork CMS can now contain Module Extension files. They
will load module specific services/configuration/event listeners in the
dependency injection container.

This is _the_ extension point for our module. It helps you create nicely
decoupled code.

I'll fill up the services.yml file later.

---

## 4. Our Add action

???

The first action I always make is the add action. It enables you to get some
data in the database, which is kind of important to test all other actions.

---

```php
<?php
namespace Backend\Modules\Team\Actions;

use Backend\Core\Engine\Model;
use Backend\Core\Engine\Base\ActionAdd;
use Backend\Modules\Team\Form\TeamType;

final class Add extends ActionAdd
{
    public function execute()
    {
        parent::execute();

        $form = new TeamType('add');
        if ($form->handle()) {
            $this->get('team_repository')->add($form->getData());

            return $this->redirect(
                Model::createURLForAction('Index') . '&report=added'
            );
        }

        $form->parse($this->tpl);
        $this->parse();
        $this->display();
    }
}
```

???

I find it important to keep my controllers (which are called actions in Fork) as
clean as possible. Controllers should only be used for these things:

* Fetching data from the request
* Communicating with services
* Sending data to the template

I don't feel building a form belongs in there, especially since the add and edit
form in Fork are mostly the same in most cases. I put it in it's own class.

---

```php
<?php
namespace Backend\Modules\Team\Repository;

use Backend\Modules\Team\Entity\TeamMember;

class TeamMemberRepository
{
    private $database;

    public function __construct(\SpoonDatabase $database)
    {
        $this->database = $database;
    }

    public function add(TeamMember $teamMember)
    {
        $teamMemberArray = $teamMember->toArray();
        $teamMemberArray['id'] = $teamMemberArray['id']->getBytes();
        $teamMemberArray['edited_on'] = $teamMemberArray['edited_on']->format('Y-m-d H:i:s');
        $teamMemberArray['created_on'] = $teamMemberArray['created_on']->format('Y-m-d H:i:s');

        return $this->database->insert(
            'team_members',
            $teamMemberArray
        );
    }
}
```

???

I don't really like the static model classes in Fork.

* They aren't testable (we can't mock)
* They access the container, when they only have some dependencies
* It's really easy to have a more clean approach.

So I create a "repository" class. The name repository comes from the repository
pattern. In this pattern, a repository is a class that is a storage system, but
behaves like a Collection of data.

The nice thing about repositories, is that you can create multiple implementations
of them. In this example, I just create a "SpoonDatabase" implementation, but
it's possible to create an Interface, with multiple implementations.

An "InMemory" version could for example be very useful for fast tests.

---

```yaml
# Make it available using $this->get('team_repository');
services:
  team_repository:
    class: Backend\Modules\Team\Repository\TeamMemberRepository
    arguments:
      - @database
```

???

We put our repository in our container by adding it in our services.yml file.
This way, we can fetch it from our container using `$this->get()`;

The Symfony DIC will nicely create an instance the first time we ask for it and
inject the database services in it as a dependency.

---

```php
<?php

namespace Backend\Modules\Team\Form;

use Backend\Core\Engine\Form;
use Backend\Modules\Team\Entity\TeamMember;

class TeamType
{
    private $form;
    private $meta;
    private $teamMember = null;

    public function __construct($name, TeamMember $teamMember = null)
    {
        $this->form = new Form($name);
        $this->teamMember = $teamMember;

        $this->build();
    }
}
```

???

Our teamtype is a form type. It accepts a name and (maybe) a teammember object.
Internally, it will build a BackendForm for us containing all the formfields we
want.

---

```php
private function build()
{
    $this->form->addText(
        'name',
        $this->teamMember === null ? null : $this->teamMember->getName(),
        null,
        'inputText title',
        'inputTextError title'
    );

    $this->form->addEditor(
        'description',
        $this->teamMember === null ? null : $this->teamMember->getDescription()
    );

    $this->meta = new Meta(
        $this->form,
        $this->teamMember === null ? null : $this->teamMember->getMetaId(),
        'name',
        true
    );
}
```

???

We add all the fields we want, plus our meta.

Attaching our data to our form fiels is a little nasty, but thats due to
SpoonFormFields not allowing to alter the data after the creation of the form.

The data can only be changed using $_POST or $_GET superglobals (dependant on
the verb of the form)

---

```php
public function handle()
{
    if (!$this->form->isSubmitted() || !$this->isValid()) {
        return false;
    }

    $fields = $this->form->getFields();

    if ($this->teamMember instanceof TeamMember) {
        $this->teamMember->change(
            $this->meta->save(),
            $fields['name']->getValue(),
            $fields['description']->getValue()
        );
        return true;
    }

    $this->teamMember = TeamMember::create(
        $this->meta->save(),
        Language::getWorkingLanguage(),
        $fields['name']->getValue(),
        $fields['description']->getValue()
    );
    return true;
}
```

???

Our handle method will change or create the teammember entity, store it internally
and return true if created/changed or false if not submitted/not valid.

---

```php
private function isValid()
{
    $fields = $this->form->getFields();

    $fields['name']->isFilled(Language::err('FieldIsRequired'));
    $fields['description']->isFilled(Language::err('FieldIsRequired'));

    $this->meta->validate();

    return $this->form->isCorrect();
}
```

???

Our handle method calls an isValid method. This method will validate all our
wanted fields, and returns if the form is correct.

I prefer "isValid" over "isCorrect", so I use another convention then SpoonForms

---

```php
public function getData()
{
    return $this->teamMember;
}

public function parse(SpoonTemplate $template)
{
    $this->form->parse($template);
}
```

???

We have just some wrapper methods left. After submitting, it's possible that
we'll do some things with our data. With the getData method, we're able to fetch
the formdata, already converted to a TeamMember entity.

The parse method just wraps the forms parse method.

---

```php
<?php

namespace Backend\Modules\Team\Engine;

use Common\Uri as CommonUri;
use Backend\Core\Engine\Language;
use Backend\Core\Engine\Model as BackendModel;

class Model
{
   public static function getUrl($url, $id = null)
    {
        $url = CommonUri::getUrl((string) $url);
        $database = BackendModel::get('database');

        // checking if it exists...

        if ($urlExists) {
            $url = Model::addNumber($url);

            return self::getUrl($url, $id);
        }

        return $url;
    }
}
```

???

The most frustrating part about this approach is the need for a Model class.
The Meta class is programmed in a way it's impossible for us to insert a service
in there instead of a static method in the model class.

I quickly make this class and never touch it again.

---

```php
<?php

namespace Backend\Modules\Team\Actions;

use Backend\Core\Engine\Base\ActionIndex;
use Backend\Core\Engine\Authentication;
use Backend\Core\Engine\DataGridDB;
use Backend\Core\Engine\DataGridFunctions;
use Backend\Core\Engine\Language;
use Backend\Core\Engine\Model;

final class Index extends ActionIndex
{
    public function execute()
    {
        parent::execute();
        $this->loadDataGrid();
        $this->parse();
        $this->display();
    }
}
```

???

SpoonDataGrid needs some more logic, so I split this controller up in methods
to keep it clean and readable.

---

```php
private function loadDataGrid()
{
    // create datagrid
    $this->dataGrid = new DataGridDB(
        $this->get('team_repository')->getDataGridQuery(),
        [ 'language' => Language::getWorkingLanguage() ]
    );

    $this->dataGrid->setColumnFunction(
        [ 'Rhumsaa\Uuid\Uuid', 'fromBytes' ],
        [ '[id]' ],
        'id',
        true
    );

    $this->dataGrid->setColumnFunction(
        [ new DataGridFunctions(), 'getLongDate' ],
        [ '[created_on]' ],
        'created_on',
        true
    );
    $this->tpl->assign('dataGrid', (string) $this->dataGrid->getContent());
}
```

???

I fetch my query from the repository. That's the place where I store all my
database related stuff, so no more class constants for me.

I convert my bytes to strings, to be able to highlight rows after add/edit.

---

```php
public function getDataGridQuery()
{
    return 'SELECT id, name, description,
                   UNIX_TIMESTAMP(created_on) AS created_on
              FROM team_members
             WHERE language = :language';
}
```

???

The getDataGridQuery method just returns a string. I use named parameters instead
of the question mark types of database parameters.

---

```php
if (Authentication::isAllowedAction('Edit')) {
    $this->dataGrid->addColumn(
        'edit',
        null,
        Language::lbl('Edit'),
        Model::createURLForAction('Edit'),
        Language::lbl('Edit')
    );
    $this->dataGrid->setColumnFunction(
        [ __CLASS__, 'addIdToEditUrl' ],
        [ '[edit]', '[id]' ],
        'edit',
        true
    );
}
```

```php
public static function addIdToEditUrl($edit, $id)
{
    return preg_replace(
        '/<a(.*)href="([^"]*)"(.*)>/',
        '<a$1href="$2' . '&amp;id=' . $id . '"$3>',
        $edit
    );
}
```

???

A second place where a little hack is required, is when parsing the id in the
Edit url. Normally, we'd just use id between square brackets. This gives us the
byte representation of the uuid though.

I just created a small column function that adds the string representation of the
uuid to the edit url

---

## Questions?

---

## Thank you!

![Thanks](img/thanks.png)

---

## Resources

<https://philsturgeon.uk/http/2015/09/03/auto-incrementing-to-destruction/>
<http://ocramius.github.io/blog/when-to-declare-classes-final/>
<http://www.fork-cms.com/blog/detail/module-specific-services>
