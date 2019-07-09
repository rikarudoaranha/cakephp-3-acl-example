# cakephp-3-acl-example
A very simple CakePHP 3 ACL plugin usage example.  This example is based on [Simple Acl controlled Application](http://book.cakephp.org/2.0/en/tutorials-and-examples/simple-acl-controlled-application/simple-acl-controlled-application.html) for CakePHP 2.  The differences are described in this document.  The files in this repository contain the changes and implementations of functions discuessed below.

### Getting started
- Assuming you are using [composer](https://getcomposer.org/), get a copy of the latest cakephp release by running `composer create-project --prefer-dist cakephp/app acl-example`.  This will create an empty CakePHP project in the `acl-example` directory.  Answer YES when asked if folder permissions should be set.
- Navigate to the CakePHP project directory (`acl-example` in this case) `cd acl-example`
- Install the [CakePHP ACL plugin](https://github.com/cakephp/acl) by running `composer require cakephp/acl`
- Include the ACL plugin in `app/config/bootstrap.php` 

```php
Plugin::load('Acl', ['bootstrap' => true]);
```

###Example schema
An example schema taken from the CakePHP 2 ACL tutorial:
```sql
CREATE TABLE users (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    password CHAR(60) NOT NULL,
    group_id INT(11) NOT NULL,
    created DATETIME,
    modified DATETIME
);

CREATE TABLE groups (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created DATETIME,
    modified DATETIME
);

CREATE TABLE posts (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id INT(11) NOT NULL,
    title VARCHAR(255) NOT NULL,
    body TEXT,
    created DATETIME,
    modified DATETIME
);

CREATE TABLE widgets (
    id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    part_no VARCHAR(12),
    quantity INT(11)
);
```
After the schema is created, proceed to "bake" the application.

```bash
bin/cake bake all groups
bin/cake bake all users
bin/cake bake all posts
bin/cake bake all widgets
```

### Preparing to Add Auth
Add `UsersController::login` function
```php
public function login() {
	if ($this->request->is('post')) {
		$user = $this->Auth->identify();
		if ($user) {
			$this->Auth->setUser($user);
			return $this->redirect($this->Auth->redirectUrl());
		}
		$this->Flash->error(__('Your username or password was incorrect.'));
	}
}
```
Add `UsersController::logout` function
```php
public function logout() {
	$this->Flash->success(__('Good-Bye'));
	$this->redirect($this->Auth->logout());
}
```
Add `src/Templates/Users/login.ctp`
```php
<?= $this->Form->create() ?>
<fieldset>
	<legend><?= __('Login') ?></legend>
	<?= $this->Form->input('username') ?>
	<?= $this->Form->input('password') ?>
	<?= $this->Form->submit(__('Login')) ?>
</fieldset>
<?= $this->Form->end() ?>
```
Modify `UsersTable::beforeSave` to hash the password before saving
```php
use Cake\Auth\DefaultPasswordHasher;
...
public function beforeSave(\Cake\Event\Event $event, \Cake\ORM\Entity $entity, 
	\ArrayObject $options)
{
	$hasher = new DefaultPasswordHasher;
	$entity->password = $hasher->hash($entity->password);
	return true;
}
```
Include and configure the `AuthComponent` and the `AclComponent` in the `AppController`
```php
public $components = [
	'Acl' => [
		'className' => 'Acl.Acl'
	]
];
...
$this->loadComponent('Auth', [
	'authorize' => [
		'Acl.Actions' => ['actionPath' => 'controllers/']
	],
	'loginAction' => [
		'plugin' => false,
		'controller' => 'Users',
		'action' => 'login'
	],
	'loginRedirect' => [
		'plugin' => false,
		'controller' => 'Posts',
		'action' => 'index'
	],
	'logoutRedirect' => [
		'plugin' => false,
		'controller' => 'Users',
		'action' => 'login'
	],
	'unauthorizedRedirect' => [
		'controller' => 'Users',
		'action' => 'login',
		'prefix' => false
	],
	'authError' => 'You are not authorized to access that location.',
	'flash' => [
		'element' => 'error'
	]
]);
```
 
### Add Temporary Auth Overrides
Temporarily allow access to `UsersController` and `GroupsController` so groups and users can be added. Add the following implementation of `beforeFilter` to `src/Controllers/UsersController.php` and `src/Controllers/GroupsController.php`:
```php
public function initialize()
{
	parent::initialize();
	
	$this->Auth->allow();
}
```  

### Initialize the Db Acl tables
- Create the ACL related tables by running `bin/cake Migrations.migrations migrate -p Acl`

### Model Setup
#### Acting as a requester
- Add the requester behavior to `GroupsTable` and `UsersTable` by adding `$this->addBehavior('Acl.Acl', ['type' => 'requester']);` to the `initialize` function in the files `src/Model/Table/UsersTable.php` and `src/Model/Table/GroupsTable.php`


#### Implement `parentNode` function in `Group` entity
Add the following implementation of `parentNode` to the file `src/Model/Entity/Group.php`:
```php
public function parentNode()
{
	return null;
}
```

#### Implement `parentNode` function in `User` entity
Add the following implementation of `parentNode` to the file `src/Model/Entity/User.php`:
```php
public function parentNode()
{
	if (!$this->id) {
		return null;
	}
	if (isset($this->group_id)) {
		$groupId = $this->group_id;
	} else {
		$Users = TableRegistry::get('Users');
		$user = $Users->find('all', ['fields' => ['group_id']])->where(['id' => $this->id])->first();
		$groupId = $user->group_id;
	}
	if (!$groupId) {
		return null;
	}
	return ['Groups' => ['id' => $groupId]];
}
```

### Creating ACOs
The [ACL Extras](https://github.com/markstory/acl_extras/) plugin referred to in the CakePHP 2 ACL tutorial is now integrated into the [CakePHP ACL plugin](https://github.com/cakephp/acl) for CakePHP 3.
- Run `bin/cake acl_extras aco_sync` to automatically create ACOs.
- ACOs and AROs can be managed manually using the ACL shell.  Run `bin/cake acl` for more information.

### Creating Users and Groups
#### Create Groups
- Navigate to `/groups/add` and add the groups
  - For this example, we will create `Administrator`, `Manager`, and `User`

#### Create Users
- Navigate to `/users/add` and add the users
  - For this example, we will create one user in each group
    - `test-administrator` is an `Administrator`
    - `test-manager` is a `Manager`
    - `test-user` is a `User`
	
### Remove Temporary Auth Overrides
Remove the temporary auth overrides by removing the `beforeFilter` function or the call to `$this->Auth->allow();` in `src/Controllers/UsersController.php` and `src/Controllers/GroupsController.php`.
	
### Configuring Permissions
#### Configuring permissions using the ACL shell
First, find the IDs of each group you want to grant permissions on.  There are several ways of doing this.  Since we will be at the console anyway, the quickest way is probably to run `bin/cake acl view aro` to view the ARO tree.  In this example, we will assume the `Administrator`, `Manager`, and `User` groups have IDs 1, 2, and 3 respectively.
- Grant members of the `Administrator` group permission to everything
  - Run `bin/cake acl grant Groups.1 controllers`
- Grant members of the `Manager` group permission to all actions in `Posts` and `Widgets`
  - Run `bin/cake acl deny Groups.2 controllers`
  - Run `bin/cake acl grant Groups.2 controllers/Posts`
  - Run `bin/cake acl grant Groups.2 controllers/Widgets`
- Grant members of the `User` group permission to view `Posts` and `Widgets`
  - Run `bin/cake acl deny Groups.3 controllers`
  - Run `bin/cake acl grant Groups.3 controllers/Posts/index`
  - Run `bin/cake acl grant Groups.3 controllers/Posts/view`
  - Run `bin/cake acl grant Groups.3 controllers/Widgets/index`
  - Run `bin/cake acl grant Groups.3 controllers/Widgets/view`
- Allow all groups to logout
  - Run `bin/cake acl grant Groups.2 controllers/Users/logout`
  - Run `bin/cake acl grant Groups.3 controllers/Users/logout`
