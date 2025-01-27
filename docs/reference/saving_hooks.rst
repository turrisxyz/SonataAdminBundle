Saving hooks
============

When a SonataAdmin is submitted for processing, there are some events called. One
is before any persistence layer interaction and the other is afterward. Also between submitting
and validating for edit and create actions ``preValidate`` event called. The
events are named as follows:

- new object : ``preValidate($object)`` / ``prePersist($object)`` / ``postPersist($object)``
- edited object : ``preValidate($object)`` / ``preUpdate($object)`` / ``postUpdate($object)``
- deleted object : ``preRemove($object)`` / ``postRemove($object)``

It is worth noting that the update events are called whenever the Admin is successfully
submitted, regardless of whether there are any actual persistence layer events. This
differs from the use of ``preUpdate`` and ``postUpdate`` events in DoctrineORM and perhaps some
other persistence layers.

For example: if you submit an edit form without changing any of the values on the form
then there is nothing to change in the database and DoctrineORM would not fire the **Entity**
class's own ``preUpdate`` and ``postUpdate`` events. However, your **Admin** class's
``preUpdate``  and  ``postUpdate`` methods *are* called and this can be used to your
advantage.

.. note::

    When embedding one Admin within another, for example using the ``sonata_type_admin``
    field type, the child Admin's hooks are **not** fired.

Example used with the FOS/UserBundle
------------------------------------

The ``FOSUserBundle`` provides authentication features for your Symfony Project,
and is compatible with Doctrine ORM, Doctrine ODM. See
`FOSUserBundle on GitHub`_ for more information.

The user management system requires to perform specific calls when the user
password or username are updated. This is how the Admin bundle can be used to
solve the issue by using the ``preUpdate`` saving hook::

    namespace Sonata\UserBundle\Admin\Entity;

    use Sonata\AdminBundle\Admin\AbstractAdmin;
    use FOS\UserBundle\Model\UserManagerInterface;
    use Sonata\AdminBundle\Form\Type\ModelType;
    use Sonata\UserBundle\Form\Type\SecurityRolesType;

    final class UserAdmin extends AbstractAdmin
    {
        protected function configureFormFields(FormMapper $form): void
        {
            $form
                ->with('General')
                    ->add('username')
                    ->add('email')
                    ->add('plainPassword', 'text')
                ->end()
                ->with('Groups')
                    ->add('groups', ModelType::class, ['required' => false])
                ->end()
                ->with('Management')
                    ->add('roles', SecurityRolesType::class, ['multiple' => true])
                    ->add('locked', null, ['required' => false])
                    ->add('expired', null, ['required' => false])
                    ->add('enabled', null, ['required' => false])
                    ->add('credentialsExpired', null, ['required' => false])
                ->end()
            ;
        }

        public function preUpdate(object $user): void
        {
            $this->getUserManager()->updateCanonicalFields($user);
            $this->getUserManager()->updatePassword($user);
        }

        public function setUserManager(UserManagerInterface $userManager): void
        {
            $this->userManager = $userManager;
        }

        public function getUserManager(): UserManagerInterface
        {
            return $this->userManager;
        }
    }

The service declaration where the ``UserManager`` is injected into the Admin class.

.. configuration-block::

    .. code-block:: xml

        <service id="fos.user.admin.user" class="%fos.user.admin.user.class%">
            <call method="setUserManager">
                <argument type="service" id="fos_user.user_manager"/>
            </call>
            <tag name="sonata.admin" model_class="%fos.user.admin.user.entity%" manager_type="orm" group="fos_user"/>
        </service>

Hooking in the Controller
-------------------------

You may have noticed that the hooks present in the **Admin** do not allow you
to interact with the process of deletion: you can't cancel it. To achieve this
you should be aware that there is also a way to hook on actions in the Controller.

If you define a custom controller that inherits from ``CRUDController``, you can
redefine the following methods:

- new object : ``preCreate($object)``
- edited object : ``preEdit($object)``
- deleted object : ``preDelete($object)``
- show object : ``preShow($object)``
- list objects : ``preList($object)``

If these methods return a **Response**, the process is interrupted and the response
will be returned as is by the controller (if it returns null, the process continues).
You can generate a redirection to the object show page by using the method ``redirectTo($object)``.

.. note::

    If you need to prohibit the deletion of a specific item, you may do a check
    in the ``preDelete($object)`` method.

.. _FOSUserBundle on GitHub: https://github.com/FriendsOfSymfony/FOSUserBundle/
