# Extensions Doctrine dans API Platform
---
__Les extensions Doctrine dans API Platform permettent de simplifier du requêtage avancé vers une BDD.__ Cela évite de créer des Controllers personnalisés.

## Fonctionnement
Le requêtage se fait grâce aux interfaces QueryCollectionExtensionInterface et QueryItemExtensionInterface qui vont permettre d'agir sur des collections et des éléments d'entités.
Le principe est celui du polymorphisme : une classe fille peut être appelée à la place d'une classe mère, grâce aux interfaces qui fixent leur structure.

## Utilisation
Nous allons prendre l'exemple d'une API qui doit permettre de retrouver les cinq dernières compositions d'un.e utilisat.eur.rice.
Soient les entités  User et Composition :
```php
<?php
// src/Entity/User.php
namespace App\Entity;
 
use ApiPlatform\Core\Annotation\ApiResource;
 
/**
 * @ApiResource
 */
class User
{
    //...
 
   /**
     * @ORM\OneToMany(targetEntity=composition::class, mappedBy="compositionsList")
     */
    private $compositionsList;

    //...
}
```

```php
<?php
// src/Entity/Composition.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
 //...

/**
 * @ApiResource()
 * @ORM\Entity(repositoryClass=CompositionRepository::class)
 */
class Composition
{
   //...
    /**
     * @ORM\ManyToOne(targetEntity=User::class, inversedBy="compositionsList")
     */
    private $user;
 //...
```
Nous créons un fichier pour mettre en place l'extension et nous l'appelons OldCompositionExtension.php.
```php
<?php
// src/Doctrine/OldCompositionExtension.php
// Adapted from the API Platform CurrentUserExtension example.
 
namespace App\Doctrine;
 
use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use App\Entity\Composition;
use App\Entity\User;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
 
final class OldCompositionExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
{
    private $tokenStorage;
    private $authorizationChecker;
 
    public function __construct(TokenStorageInterface $tokenStorage, AuthorizationCheckerInterface $checker)
    {
        $this->tokenStorage = $tokenStorage;
        $this->authorizationChecker = $checker;
    }
 
    /**
     * {@inheritdoc}
     */
    public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }
 
    /**
     * {@inheritdoc}
     */
    public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }
 
    /**
     *
     * @param QueryBuilder $queryBuilder
     * @param string       $resourceClass
     */
    private function addWhere(QueryBuilder $queryBuilder, string $resourceClass)
    {
        $user = $this->tokenStorage->getToken()->getUser();
 
        // Add the where clause if we're operating on a Composition resource, and the user is not an admin.
        if ($user instanceof User && Composition::class === $resourceClass && count($compositionList) >= 5) {
            $rootAlias = $queryBuilder->getRootAliases()[0];
            $lastCompositionList = [];
            for ($i = 0; $i <= 5; $i++) {
                array_push($lastCompositionList, $compositionList[i]);
                return $lastCompositionList;
            }
            $queryBuilder->andWhere(sprintf('%s.$lastCompositionList', $rootAlias));
        }
    }
}
```
2. L'auto-configuration est active car dans le fichier service, nous avons la ligne suivante :
```yaml
# api/config/services.yaml
services:
    _defaults:
...
        autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
```