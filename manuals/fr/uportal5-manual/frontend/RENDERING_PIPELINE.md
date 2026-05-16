# Configuration du pipeline de rendu d'uPortal

uPortal implémente le rendu complet de pages en utilisant un _pipeline_: une structure imbriquée d'éléments discrets,
pluggables, qui implémentent chacun la même interface Java. Le mot "pipeline" est approprié
parce qu'il invoque les concepts de _mouvement_ et de _flux_, mais dans le logiciel, cette conception est également
connu sous le nom [Decorator pattern][].

L'interface Java au centre du pipeline de rendu uPortal est `IPortalRenderingPipeline`.
Les instances de `IPortalRenderingPipeline` sont des beans gérés par Spring. Le bean primaire du pipeline du rendu
reçoit un identifiant (dans Spring) de `portalRenderingPipeline`. Les composants d'autres parties du
portail (en dehors du pipeline de rendu) utilisent ce bean (exclusivement) pour interagir avec le rendu dans
le portail.

L'interface `IPortalRenderingPipeline` ne définit qu'une seule méthode :

```java
public void renderState(HttpServletRequest req, HttpServletResponse res)
        throws ServletException, IOException;
```

## Pipeline Standard uPortal

Le pipeline de rendu "standard" est celui qui vient avec uPortal prêt à l'emploi: par défaut,
uPortal 5 utilise la même configuration de pipeline de rendu que uPortal 4.3 - basée sur une instance de
`DynamicRenderingPipeline` qui contient un nombre de _composants_.

Les composants de `DynamicRenderingPipeline` implémentent chacun une interface unique:
`CharacterPipelineComponent`, qui lui-même étend
`PipelineComponent <CharacterEventReader, CharacterEvent>`. (Le pipeline de rendu standard est relié à XML / XSLT.)
Chaque composant implémente une étape discrète dans le rendu d'une demande de page.

Le pipeline standard comprend (à ce jour) les composants suivants (étapes):

1. `analyticsIncorporationComponent`
2. `portletRenderingIncorporationComponent`
3. `portletRenderingInitiationCharacterComponent`
4. `themeCachingComponent`
5. `postSerializerLogger`
6. `staxSerializingComponent`
7. `postThemeTransformLogger`
8. `themeTransformComponent`
9. `preThemeTransformLogger`
10. `themeAttributeIncorporationComponent`
11. `portletRenderingInitiationComponent`
12. `structureCachingComponent`
13. `postStructureTransformLogger`
14. `structureTransformComponent`
15. `preStructureTransformLogger`
16. `structureAttributeIncorporationComponent`
17. `portletWindowAttributeIncorporationComponent`
18. `dashboardWindowStateSettingsStAXComponent`
19. `postUserLayoutStoreLogger`
20. `userLayoutStoreComponent`

L'ordre de traitement pour ces composants de pipeline est essentiellement _rétroactif_: du bas vers le haut.

## Utiliser les Beans `RenderingPipelineBranchPoint`

Les intégrateurs d'uPortal peuvent configurer le pipeline de rendu en fonction de leur besoin. La plupart des cas d'utilisation
peuvent être satisfaits en utilisant les Beans `RenderingPipelineBranchPoint`. les _Rendering branch points_ sont des
Object Java (Beans gérés par Spring) qui indiquent à quelques (ou toutes les) requêtes HTTP de suivre un chemin different.
Les _Rendering branch points_ (points de branchement de rendu) suivent la stratégie de configuration uPortal 5 standard pour des Beans gérés par Spring :
si vous fournissez un Bean correctement configuré du type correct (_viz._
`RenderingPipelineBranchPoint`) au contexte d'application Spring, uPortal le _découvrira_ et
_l'appliquera_. (uPortal le fournira comme une dépendance aux composants qui sauront quoi
faire avec.)

uPortal évalue les Beans `RenderingPipelineBranchPoint`, si présent, dans un ordre spécifique. Si une
branche indique qu'elle _doit_ être suivie, il _va_ la suivre et aucune autre branche ne sera
testée. Si aucune branche n'est suivie, le pipeline de rendu standard sera utilisé.

Le Beans `RenderingPipelineBranchPoint` acceptent les paramètres de configuration suivant :

| Propriétés      | Type                                               | Requis ? | Notes                                                                                                                                                                                                  |
| --------------- | -------------------------------------------------- | :------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `order`         | `int`                                              |    N°    | Définit la séquence des points de branchement lorsqu'il y en a plusieurs (dans ce cas, `order` est requis). Les branches avec des valeurs d'ordre inférieures viennent avant les valeurs plus élevées. |
| `predicate`     | `java.util.function.Predicate<HttpServletRequest>` |    O     | Si le `predicate` renvoie `true`, la branch sera suivie; sinon la branche suivante sera testée.                                                                                                        |
| `alternatePipe` | `IPortalRenderingPipeline`                         |    O     | Le chemin de rendu qui sera suivi si le `predicate` renvoie` true`..                                                                                                                                   |

### Exemples

Les exemples suivants illustrent certaines utilisations typiques des beans `RenderingPipelineBranchPoint`. Chacun
de ces exemples peuvent être configurés dans

`uPortal-start/overlays/uPortal/src/main/resources/properties/contextOverrides/overridesContext.xml`.

#### Exemple 1: Rediriger les utilisateurs non authentifiés vers CAS/Shibboleth

Cet exemple illustre une fonctionnalité couramment demandée: interdire l'accès non authentifié au
portail.

```xml
<bean id="guestUserBranchPoint" class="org.apereo.portal.rendering.RenderingPipelineBranchPoint">
    <property name="predicate">
        <bean class="org.apereo.portal.rendering.predicates.GuestUserPredicate" />
    </property>
    <property name="alternatePipe">
        <bean class="org.apereo.portal.rendering.RedirectRenderingPipelineTerminator">
            <property name="redirectTo" value="${org.apereo.portal.channels.CLogin.CasLoginUrl}" />
        </bean>
    </property>
</bean>
```

#### Exemple 2: Intégration de uPortal-home

Cet exemple illustre la configuration requise du Pipeline de rendu uPortal pour l'intégration avec
[uPortal-home][].

```xml
<bean id="redirectToWebMaybe" class="org.jasig.portal.rendering.RenderingPipelineBranchPoint">
    <property name="order" value="1" />
    <property name="predicate">
        <bean class="org.jasig.portal.rendering.predicates.GuestUserPredicate" />
    </property>
    <property name="alternatePipe" ref="redirectToWeb" />
</bean>

<bean id="maybeRedirectToExclusive" class="org.jasig.portal.rendering.RenderingPipelineBranchPoint">
    <property name="order" value="2" />
    <property name="predicate">
        <bean class="java.util.function.Predicate" factory-method="and" factory-bean="focusedOnOnePortletPredicate">
            <constructor-arg>
                <bean class="java.util.function.Predicate" factory-method="and" factory-bean="urlNotInExclusiveStatePredicate">
                    <constructor-arg>
                        <bean class="org.jasig.portal.rendering.predicates.RenderOnWebFlagSetPredicate" />
                    </constructor-arg>
                </bean>
            </constructor-arg>
        </bean>
    </property>
    <property name="alternatePipe" ref="redirectToWebExclusive" />
</bean>

<!-- if the request is for a simple content portlet, redirect to
     uPortal-home to render that portlet statically. -->
<bean id="maybeRedirectToWebStatic" class="org.jasig.portal.rendering.RenderingPipelineBranchPoint">
    <property name="order" value="3" />
    <property name="predicate">
        <bean class="java.util.function.Predicate" factory-method="and" factory-bean="focusedOnOnePortletPredicate">
            <constructor-arg>
                <bean class="java.util.function.Predicate" factory-method="and" factory-bean="urlInMaximizedStatePredicate">
                    <constructor-arg>
                        <ref bean="webAppNameContainsSimpleContentPortletPredicate" />
                    </constructor-arg>
                </bean>
            </constructor-arg>
        </bean>
    </property>
    <property name="alternatePipe" ref="redirectToWebStatic" />
</bean>

<!-- Common Predicates -->

<bean id="focusedOnOnePortletPredicate" class="org.jasig.portal.rendering.predicates.FocusedOnOnePortletPredicate" />

<bean id="urlNotInExclusiveStatePredicate" class="org.jasig.portal.rendering.predicates.URLInSpecificStatePredicate">
    <property name="state" value="EXCLUSIVE" />
    <property name="negated" value="true" />
</bean>

<bean id="urlInMaximizedStatePredicate" class="org.jasig.portal.rendering.predicates.URLInSpecificStatePredicate">
    <property name="state" value="MAX" />
</bean>

<bean id="webAppNameContainsSimpleContentPortletPredicate" class="org.jasig.portal.rendering.predicates.WebAppNameContainsStringPredicate">
    <property name="webAppNameToMatch" value="SimpleContentPortlet" />
</bean>

<!-- Pipeline Terminators -->

<bean id="redirectToWeb" class="org.jasig.portal.rendering.RedirectRenderingPipelineTerminator">
    <property name="redirectTo" value="${angular.landing.page}" />
</bean>

<bean id="redirectToWebExclusive" class="org.jasig.portal.rendering.RedirectRenderingPipelineTerminator">
    <property name="redirectTo" value="${angular.landing.page}exclusive/" />
    <property name="appender" value="fname" />
</bean>

<!-- Redirect to uPortal-home,
instructing uPortal-home to render a particular portlet statically. -->
<bean id="redirectToWebStatic" class="org.jasig.portal.rendering.RedirectRenderingPipelineTerminator">
    <property name="redirectTo" value="${angular.landing.page}static/" />
    <property name="appender" value="fname" />
</bean>
```

[Decorator pattern]: https://en.wikipedia.org/wiki/Decorator_pattern
[uPortal-home]: https://github.com/uPortal-Project/uportal-home
