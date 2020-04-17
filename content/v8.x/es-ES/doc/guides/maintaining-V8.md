# Mantenimiento de V8 en Node.js

## Trasfondo

V8 sigue el programa de lanzamiento de Chromium. The support horizon for Chromium is different compared to the support horizon for Node.js. As a result, Node.js needs to support multiple versions of V8 longer than what upstream needs to support. V8 branches in Node.js lack of an official maintenance process due to a missing LTS supported branch.

This document attempts to outline the current maintenance processes, proposes a workflow for maintaining the V8 branches in both Node.js LTS and current releases, and discusses how the Node.js and V8 teams at Google can help.

## Programa de Lanzamiento de V8

V8 y Chromium siguen una [cadencia de lanzamiento de aproximadamente 6 semanas](https://www.chromium.org/developers/calendar). En cualquier momento dado hay tres ramas de V8 que están **activas**.

Por ejemplo, al momento de escribir esto:

* **Estable**: V8 5.4 se envía actualmente como parte de una versión estable de Chromium. Esta rama fue creada aprox. 6 semanas antes desde que V8 5.3 se enviara como estable.
* **Beta**: V8 5.5 está actualmente en beta. Será promovido a estable luego; aproximadamente 6 semanas después del envío de V8 5.4 como estable.
* **Master**: V8 tip-of-tree corresponde a V8 5.6. Esta rama se lanza regularmente como parte de las compilaciones Chromium **canary**. Se promoverá a beta luego, cuando V8 5.5 se envíe como una versión estable.

Todas las ramas más antiguas son abandonadas y no son mantenidas por el equipo de V8.

### Descripción general del proceso de fusión V8

El proceso para respaldar las correcciones de errores en las ramas activas está oficialmente documentado [en la wiki de V8](https://github.com/v8/v8/wiki/Merging%20&%20Patching). El resumen del proceso es:

* V8 sólo soporta ramas activas. No se realizan pruebas en ramas anteriores a la estable/beta/master actual.
* Una corrección que necesite un backport se etiqueta con la etiqueta *merge-request-x.x*. Esto puede hacerlo cualquier persona interesada en que se le haga backport a la solución. Los problemas con esta etiqueta son revisados por el equipo de V8 regularmente como candidatos para backporting.
* Las correcciones necesitan cierto "tiempo de cocción" antes de que puedan ser aprobadas para backporting. Esto significa esperar unos días para asegurar que no se detecten problemas en las compilaciones canary/beta.
* Una vez que esté listo, el problema es etiquetado con *merge-approved-x.x* y uno puede hacer la fusión real utilizando los scripts en la [página wiki](https://github.com/v8/v8/wiki/Merging%20&%20Patching).
* Las solicitudes de fusión a una rama abandonada serán rechazadas.
* Sólo se aceptan las correcciones de errores para backporting.

## Requerimientos para Soporte de Node.js

En un momento dado, Node.js necesita mantener algunas ramas de V8 diferentes para las distintas versiones actuales, LTS y lanzamientos nocturnos. En la actualidad, esta lista incluye las siguientes ramas <sup>1</sup>:

<table>
  <tr>
   <td><strong>Lanzamiento</strong>
   </td>
   <td><strong>Inicio de Soporte</strong>
   </td>
   <td><strong>Fin de Soporte</strong>
   </td>
   <td><strong>Versión V8</strong>
   </td>
   <td><strong>Rama de V8 lanzada</strong>
   </td>
   <td><strong>Rama de V8 abandonada</strong>
   </td>
  </tr>
  <tr>
   <td>Node.js 4.x
   </td>
   <td>01-10-2015
   </td>
   <td>Abril 2018
   </td>
   <td>4.5
   </td>
   <td>01-09-2015
   </td>
   <td>13-10-2015
   </td>
  </tr>
  <tr>
   <td>Node.js 6.x
   </td>
   <td>01-04-2016
   </td>
   <td>Abril 2019
   </td>
   <td>5.1
   </td>
   <td>31-05-2016
   </td>
   <td>26-06-2016
   </td>
  </tr>
  <tr>
   <td>Node.js 8.x
   </td>
   <td>30-05-2017
   </td>
   <td>Diciembre 2019
   </td>
   <td>6.1 (pronto a ser 6.2)
   </td>
   <td>17-10-2017 (6.2)
   </td>
   <td>~05-12-2017 (6.2)
   </td>
  </tr>
    <tr>
   <td>Node.js 9.x
   </td>
   <td>31-10-2017
   </td>
   <td>Abril 2018
   </td>
   <td>6.2
   </td>
   <td>17-10-2017
   </td>
   <td>~05-12-2017
   </td>
  </tr>
  <tr>
   <td>master
   </td>
   <td>N/A
   </td>
   <td>N/A
   </td>
   <td>6.2
   </td>
   <td>17-10-2017
   </td>
   <td>~05-12-2017
   </td>
  </tr>
</table>

Las versiones de V8 utilizadas en Node.js v4.x, v6.x y 8.x ya han sido abandonadas por V8 upstream. Sin embargo, Node.js necesita continuar apoyando estas ramas durante muchos meses (ramas actuales), o por varios años (ramas LTS).

## Proceso de Mantenimiento

Una vez que se ha identificado que una falla en Node.js es causada por V8, el primer paso es identificar las versiones de Node.js y V8 afectadas. El error puede estar presente en múltiples ubicaciones diferentes, cada una de las cuales sigue un proceso ligeramente diferente.

* Errores sin corregir. El error existe en la rama master de V8.
* Corregido, pero necesita backport. El error puede necesitar la transferencia a una o más ramas. 
    * Backporting a ramas activas.
    * Backporting a ramas abandonadas.
* Backports identificados por el equipo de V8. Errores identificados por el upstream V8 que no hemos encontrado en Node.js todavía.

### Errores de Upstream No Corregidos

Si el error se puede reproducir en la [rama `vee-eight-lkgr` ](https://github.com/v8/node/tree/vee-eight-lkgr), Chromium canary o V8 tip-of-tree, y el caso de prueba es válido, entonces el error debe corregirse upstream primero.

* Comience abriendo un upstream de error [usando esta plantilla](https://bugs.chromium.org/p/v8/issues/entry?template=Node.js%20upstream%20bug).
* Asegúrese de incluir un enlace al problema de Node.js correspondiente (si existe).
* Si la corrección es lo suficientemente simple, puede corregirlo usted mismo; las [contribuciones](https://github.com/v8/v8/wiki/Contributing) son bienvenidas.
* La cascada de compilación de V8 prueba su cambio.
* Una vez que se haya corregido el error, es posible que todavía se necesite backporting, si existe en otras ramas de V8 que todavía estén activas o son ramas que le importan a Node.js. Siga el proceso de backporting a continuación.

### Backporting a Ramas Activas

Si el error existe en cualquiera de las ramas activas de V8, es posible que necesitemos hacer backport a la solución. En cualquier momento dado hay [dos ramas activas](https://build.chromium.org/p/client.v8.branches/console) (beta y stable), además de la master. Los siguientes pasos son necesarios para respaldar la corrección:

* Identificar en qué versión de V8 se corrigió el error.
* Identificar si alguna rama de V8 activa todavía contiene el error:
* Se necesita un error de seguimiento para solicitar un backport. 
    * Si todavía no hay un error V8 que rastree la corrección, abra un nuevo error de solicitud de fusión usando esta [plantilla específica de Node.js](https://bugs.chromium.org/p/v8/issues/entry?template=Node.js%20merge%20request).
    * Si ya existe un error 
        * Añadir una referencia al problema de GitHub.
        * Adjuntar las etiquetas *merge-request-x.x* al error para cualquier rama activa que todavía contenga el error. (por ejemplo, merge-request-5.3, merge-request-5.4)
        * Agregue ofrobots-at-google.com a la lista cc.
* Una vez que la fusión haya sido aprobada, debe fusionarse utilizando el [script de fusión documentado en el wiki de V8](https://github.com/v8/v8/wiki/Merging%20&%20Patching). La fusión requiere tener acceso de commits al repositorio de V8. Si no tiene acceso de commits puede indicar que alguien en el equipo de V8 haga la fusión por usted.
* Es posible que la solicitud de fusión no sea aprobada, por ejemplo, si se considera que es una característica o si es demasiado arriesgada para una versión estable de V8. En tales casos, flotamos el parche en el lado de Node.js. Vea el proceso sobre 'Backporting a ramas Abandonadas'.
* Una vez que la solución se ha fusionado upstream, se puede recoger durante una actualización de la rama V8 (ver a continuación).

### Backporting a Ramas Abandonadas

Las ramas de V8 abandonadas son soportadas en el repositorio de V8 de Node.js. La corrección debe ser seleccionada en el repositorio Node.js y V8-CI debe probar el cambio.

* Para cada rama de V8 abandonada correspondiente a una rama LTS que se ve afectada por el error: 
    * Abra una PR de cherry-pick en nodejs/node seleccionando la rama *vY.x-staging* apropiada (por ejemplo, *v6.x-staging* para solucionar un problema en V8-5.1).
    * Incremente la versión de nivel de parche en `v8-version.h`. Esto no causará ningún problema con el control de versiones porque V8 no publicará otros parches para esta rama, por lo que Node.js puede efectivamente superar la versión del parche.
    * En algunos casos, el parche puede requerir un esfuerzo extra para fusionarse en caso de que V8 haya cambiado substancialmente. Para problemas importantes, es posible que podamos apoyarnos en el equipo de V8 para obtener ayudar con la reimplementación del parche.
    * Ejecute el [V8-CI](https://ci.nodejs.org/job/node-test-commit-v8-linux/) de Node.js además del [Node.js CI](https://ci.nodejs.org/job/node-test-pull-request/).

An example for workflow how to cherry-pick consider the bug [RegExp show inconsistent result with other browsers](https://crbug.com/v8/5199). From the bug we can see that it was merged by V8 into 5.2 and 5.3, and not into V8 5.1 (since it was already abandoned). Since Node.js `v6.x` uses V8 5.1, the fix needed to be cherry-picked. Para seleccionarla cuidadosamente, aquí hay un ejemplo de flujo de trabajo:

* Descargue y aplique el commit vinculado en el problema (en este caso a51f429). `curl -L https://github.com/v8/v8/commit/a51f429.patch | git am -3 --directory=deps/v8`. Si las ramas han divergido significativamente, esto puede no aplicarse limpiamente. Puede ser útil tratar de hacer cherry-pick a la fusión a la rama más antigua que se realizó upstream en V8. En este ejemplo, este sería el parche de la fusión a 5.2. La esperanza es que esto esté más cerca del V8 5.1 y que tenga una mejor oportunidad de aplicar limpiamente. Si está atascado, siéntase libre de hacer ping a @ofrobots para ayuda.
* Modifique el mensaje de commit para que coincida con el formato que utilizamos para backports de V8 y colóquese como autor. `git commit --amend --reset-author`. Es posible que desee agregar una descripción adicional si es necesario indicar el impacto de la corrección en Node.js. En este caso, el problema original fue lo suficientemente descriptivo. Ejemplo:

```console
deps: hacer cherry-pick a a51f429 desde upstream de V8

Original commit message:
  [regexp] Corregir la coincidencia entre mayúsculas y minúsculas para los casos de un byte.

  El error se produce porque no canonizamos los rangos de clase de caracteres
  antes de agregar equivalentes de casos. Al agregar equivalentes de casos, abortamos temprano para strings de casos de un byte, asumiendo que los rangos están ordenados.
  Que no lo están.

  R=marja@chromium.org
  BUG=v8:5199

  Review-Url: https://codereview.chromium.org/2159683002
  Cr-Commit-Position: refs/heads/master@{#37833}

Refs: https://github.com/v8/v8/commit/a51f429772d1e796744244128c9feeab4c26a854
PR-URL: https://github.com/nodejs/node/pull/7833
```

* Abra una PR contra la rama `v6.x-staging` en el repositorio de Node.js. Inicie el normal y el [V8-CI](https://ci.nodejs.org/job/node-test-commit-v8-linux/) utilizando el sistema CI de Node.js. Sólo necesitábamos hacer backport a `v6.x` ya que las otras ramas de LTS no fueron afectadas por este error.

### Backports Identificados por el equipo de V8

Para los errores encontrados a través del navegador u otros canales, el equipo V8 marca los errores que podrían ser aplicables a las ramas abandonadas en uso por Node.js. Esto se hace a través del etiquetado manual por parte del equipo de V8 y mediante un proceso automatizado que marca cualquier corrección a la que se le haya hecho un backport a la rama estable (ya que es probable que sea una candidata para el backporting).

Estas correcciones se marcan con las siguientes etiquetas en el rastreador de problemas de V8:

* `NodeJS-Backport-Review` ([V8](https://bugs.chromium.org/p/v8/issues/list?can=1&q=label%3ANodeJS-Backport-Review), [Chromium](https://bugs.chromium.org/p/chromium/issues/list?can=1&q=label%3ANodeJS-Backport-Review)): se revisará si esto es aplicable a las ramas abandonadas en uso por Node.js. Esta lista es revisada regularmente por el equipo de Node.js en Google para determinar la aplicabilidad a Node.js.
* `NodeJS-Backport-Approved` ([V8](https://bugs.chromium.org/p/v8/issues/list?can=1&q=label%3ANodeJS-Backport-Approved), [Chromium](https://bugs.chromium.org/p/chromium/issues/list?can=1&q=label%3ANodeJS-Backport-Approved)): marca errores que se consideran relevantes para Node.js y que deberían ser backported.
* `NodeJS-Backport-Done` ([V8](https://bugs.chromium.org/p/v8/issues/list?can=1&q=label%3ANodeJS-Backport-Done), [Chromium](https://bugs.chromium.org/p/chromium/issues/list?can=1&q=label%3ANodeJS-Backport-Done)): El backport para Node.js ya ha sido realizado.
* `NodeJS-Backport-Rejected` ([V8](https://bugs.chromium.org/p/v8/issues/list?can=1&q=label%3ANodeJS-Backport-Rejected), [Chromium](https://bugs.chromium.org/p/chromium/issues/list?can=1&q=label%3ANodeJS-Backport-Rejected)): No se desea un backport para Node.js.

El equipo de node en Google regularmente revisa la acumulación de problemas con estos para supervisar el proceso de backport. Los colaboradores externos también pueden colaborar en el proceso de backport. Tenga en cuenta que algunos de los errores pueden ser problemas de seguridad y no serán visibles para los colaboradores externos.

## Actualizando V8

Node.js mantiene una copia vendida de V8 dentro del directorio deps/. Además, es posible que Node.js tenga que flotar parches que no existen upstream. Esto significa que se debe tener cuidado al actualizar la copia vendida de V8.

### Actualizaciones menores (a nivel de parche)

Debido a que puede haber parches flotantes en la versión de V8 en Node.js, es más seguro aplicar las actualizaciones de nivel de parche como un parche. Por ejemplo, imagine que el V8 upstream está en 5.0.71.47 y Node.js está en 5.0.71.32. Sería mejor calcular la diferencia entre estas etiquetas en el repositorio de V8 y luego aplicar ese parche en la copia de V8 en Node.js. Esto debería preservar los parches/backports que Node.js puede estar flotando (o de lo contrario causaría un conflicto de fusión).

El bosquejo del proceso es:

```shell
# Suponiendo que su bifurcación de Node.js esté verificada en $NODE_DIR
# y desea actualizar la rama maestra Node.js.
# Encuentre la versión actual (ANTIGUA) en
# $NODE_DIR/deps/v8/include/v8-version.h
cd $NODE_DIR
git checkout master
git merge --ff-only origin/master
git checkout -b V8_NEW_VERSION
curl -L https://github.com/v8/v8/compare/${V8_OLD_VERSION}...${V8_NEW_VERSION}.patch | git apply --directory=deps/v8
# Es posible que desee modificar el mensaje de commit para describir la naturaleza de la actualización
```

V8 también mantiene etiquetas de la forma *5.4-lkgr* que apuntan a la *Última Revisión Válida Conocida* de la rama 5.4 que puede ser útil en el proceso de actualización anterior.

### Actualizaciones Principales

Actualizamos la versión de V8 en Node.js master cada vez que una versión de V8 se estabiliza upstream, es decir, cada vez que sale una nueva versión de Chrome.

Actualizar las versiones principales sería mucho más difícil de hacer con el mecanismo de parche anterior. Una mejor estrategia es

1. Audite la rama maestra actual y observe los parches que se han flotado desde la última actualización importante de V8.
2. Reemplace la copia de V8 en Node.js con una nueva comprobación de la última rama de V8 estable. Se debe tener un cuidado especial para actualizar recursivamente los DEPS en los que V8 tiene una dependencia de tiempo de compilación (al momento de escribir esto, estos son solo trace_event y gtest_prod.h)
3. Renovar (hacer cherry-pick) todos los parches de la lista calculados en 1) según sea necesario. Es posible que algunos de los parches ya no sean necesarios.

Para auditar los parches flotantes:

```shell
git log --oneline deps/v8
```

To replace the copy of V8 in Node.js, use the `[update-v8](https://gist.github.com/targos/8da405e96e98fdff01a395bed365b816)` script<sup>2</sup>. Por ejemplo, si desea reemplazar la copia de V8 en Node.js con el branch-head para la rama 5.1 de V8:

```shell
cd $NODE_DIR
rm -rf deps/v8
path/to/update-v8 branch-heads/5.1
```

Es posible que desee ver los commits creados por los scripts anteriores y aplastarlos una vez que los haya revisado.

Esto debe ser seguido con la renovación manual de todos los parches relevantes.

## Propuesta: Usar un repositorio fork para rastrear V8 upstream

El hecho de que Node.js mantenga una copia vendida y potencialmente editada de V8 en deps/ hace que los procesos anteriores sean un poco complicados. An alternative proposal would be to create a fork of V8 at `nodejs/v8` that would be used to maintain the V8 branches. Esto tiene varios beneficios:

* The process to update the version of V8 in Node.js could be automated to track the tips of various V8 branches in `nodejs/v8`.
* Simplificaría el cherry-picking y el porting de correcciones entre ramas, ya que los saltos de versión en `v8-version.h` ocurrirían como parte de esta actualización en lugar de en cada cambio.
* Simplificaría el V8-CI y lo haría más automatizable.
* The history of the V8 branch in `nodejs/v8` becomes purer and it would make it easier to pull in the V8 team for help with reviewing.
* Haría más fácil la configuración de una compilación automatizada que rastree la compilación de integración de Node.js master + V8 lkgr.

Esto requeriría de algunas herramientas para:

* Un script que actualizaría el V8 en una rama especifica de Node.js con V8 del upstream (dependiente de la rama abandonada vs. activa).
* We need a script to bump V8 version numbers when a new version of V8 is promoted from `nodejs/v8` to `nodejs/node`.
* Habilitar la compilación de V8-CI en Jenkins para compilar desde el fork `nodejs/v8`.

## Propuesta: Hacer frente a la necesidad de hacer flotar los parches a una fase estable/beta

A veces, el V8 upstream puede no querer fusionar un arreglo a sus ramas estables, pero podríamos. Un ejemplo de esto sería una corrección para una regresión de rendimiento que solo afecta a Node.js y no al navegador. Por el momento no tenemos un mecanismo para enfrentar esta situación. Si hacemos flotar un parche y se mejora la versión de V8, podríamos encontrarnos con un problema si la versión anterior libera una corrección con el mismo número de versión.

Una idea que hemos estado considerando es que podríamos pasar a un número de versión de 5 lugares en V8, por ejemplo: 5.4.500.30. ${embedder}. El ${embedder} representa la cantidad de parches que un embedder está flotando sobre una versión oficial de V8. Esto también ayudaría con la auditoría de los parches flotantes en el historial de commits de Node.js.

Estamos probando esto en https://github.com/nodejs/node/pull/9754. Si esto termina funcionando, investigaremos cómo hacer este cambio upstream.

<!-- Footnotes themselves at the bottom. -->

### Notas

<sup>1</sup>Node.js 0.12 y anteriores son omitidas intencionalmente de este documento ya que su soporte está por terminar.

<sup>2</sup>@targos está trabajando en [un puerto para este script](https://github.com/targos/update-v8).