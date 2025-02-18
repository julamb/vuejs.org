---
title: Fonctions de rendu et JSX
type: guide
order: 303
---

## Bases

Vue vous recommande l'utilisation de templates pour construire votre HTML dans la grande majorité des cas. Il y a cependant des situations où vous aurez réellement besoin de toute la puissance programmatique de JavaScript. C'est là que vous pouvez utiliser les **fonctions de rendu**, une alternative aux templates qui est plus proche du compilateur.

Examinons un exemple simple où une fonction `render` serait plus pratique. Imaginons que nous souhaitons générer des titres avec une ancre :

``` html
<h1>
  <a name="hello-world" href="#hello-world">
    Hello world !
  </a>
</h1>
```

Pour le HTML ci-dessus, vous décidez d'utiliser cette interface de composant :

``` html
<anchored-heading :level="1">Hello world!</anchored-heading>
```

Quand vous commencez avec un composant se basant sur la prop `level` pour simplement générer des niveaux de titre, vous arrivez rapidement à cela :

``` html
<script type="text/x-template" id="anchored-heading-template">
  <h1 v-if="level === 1">
    <slot></slot>
  </h1>
  <h2 v-else-if="level === 2">
    <slot></slot>
  </h2>
  <h3 v-else-if="level === 3">
    <slot></slot>
  </h3>
  <h4 v-else-if="level === 4">
    <slot></slot>
  </h4>
  <h5 v-else-if="level === 5">
    <slot></slot>
  </h5>
  <h6 v-else-if="level === 6">
    <slot></slot>
  </h6>
</script>
```

``` js
Vue.component('anchored-heading', {
  template: '#anchored-heading-template',
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

Ce template ne semble pas génial. Il n'est pas uniquement verbeux, il duplique `<slot></slot>` dans tous les niveaux de titre et nous allons devoir refaire la même chose quand nous ajouterons l'élément ancre.

Alors que les templates fonctionnent bien pour la plupart des composants, il est clair que celui-là n'est pas l'un d'entre eux. Aussi essayons de le réécrire avec une fonction `render` :

``` js
Vue.component('anchored-heading', {
  render: function (createElement) {
    return createElement(
      'h' + this.level,   // nom de balise
      this.$slots.default // tableau des enfants
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

C'est bien plus simple ! Le code est plus court mais demande une plus grande familiarité avec les propriétés d'une instance de Vue. Dans ce cas, vous devez savoir que lorsque vous passez des enfants sans une directive `v-slot` dans un composant, comme le `Hello world !` à l'intérieur de `anchored-heading`, ces enfants sont stockés dans l'instance du composant via la propriété `$slots.default`. Si vous ne l'avez pas encore fait, **il est recommandé d'en lire plus sur les [propriétés d'instance de l'API](../api/#Proprietes-dinstance) avant d'entrer plus en profondeur dans les fonctions de rendu.**

## Nœuds, arbres, et DOM virtuel

Avant de rentrer en profondeur dans les fonctions de rendu, il est important de savoir comment un navigateur fonctionne. Prenons cet HTML comme exemple :

```html
<div>
  <h1>Mon titre</h1>
  Divers contenus
  <!-- À faire : ajouter une balise -->
</div>
```

Quand votre navigateur lit ce code, il construit un [arbre de nœud de DOM](https://javascript.info/dom-nodes) pour l'aider à garder une trace de tout, comme vous ferriez un arbre généalogique pour garder une trace de votre famille étendue.

L'arbre des nœuds du DOM pour le HTML ci-dessus ressemblerait à cela :

![Visualisation de l'arbre du DOM](/images/dom-tree.png)

Chaque élément est un nœud. Chaque morceau de texte est un nœud. Même les commentaires sont des nœuds ! Un nœud est juste un morceau de la page. Et comme dans un arbre généalogique, chaque nœud peut avoir des enfants (c.-à-d. que chaque morceau peut contenir d'autres morceaux).

Mettre à jour tous ces nœuds efficacement peut être difficile, mais heureusement, vous n'avez jamais à le faire manuellement. Vous avez juste à dire à Vue quel HTML vous voulez pour votre page dans un template :

```html
<h1>{{ blogTitle }}</h1>
```

Ou quelle fonction de rendu :

``` js
render: function (createElement) {
  return createElement('h1', this.blogTitle)
}
```

Et dans les deux cas, Vue va automatiquement garder la page à jour, même quand `blogTitle` change.

### DOM virtuel

Vue arrive à cela grâce à la construction d'un **DOM virtuel** pour garder les traces des changements qui doivent être faits sur le vrai DOM. Prêtons attention à cette ligne :

``` js
return createElement('h1', this.blogTitle)
```

Qu'est-ce que `createElement` retourne exactement ? Ce n'est pas _réellement_ un vrai élément de DOM. Cela pourrait être nommé plus justement `createNodeDescription`, car il contient des informations décrivant à Vue quelle sorte de rendu de nœud il va falloir faire sur la page en incluant les descriptions des nœuds enfants. Nous appelons cette description de nœud un « nœud virtuel », usuellement abrégé en **VNode** (pour « virtual node »). « DOM virtuel » est le nom de l'arbre des VNodes construits par un arbre de composants Vue.

## Arguments de `createElement`

La seconde chose à laquelle vous allez devoir vous familiariser est la manière d'utiliser les fonctionnalités des templates avec la fonction `createElement`. Voici les arguments que la fonction `createElement` accepte :

``` js
// @returns {VNode}
createElement(
  // {String | Object | Function}
  // Un nom de balise HTML, des options de composant, ou une fonction
  // asynchrone retournant l'un des deux. Requis.
  'div',

  // {Object}
  // Un objet de données correspondant aux attributs
  // que vous souhaitez utiliser dans le template. Optionnel.
  {
    // (vu en détail dans la prochaine section)
  },

  // {String | Array}
  // Des VNodes enfants, construit en utilisant `createElement()`,
  // ou en utilisant des chaine de caractère pour créer des 'text VNodes'. Optionnel.
  [
    'Some text comes first.',
    createElement('h1', 'A headline'),
    createElement(MyComponent, {
      props: {
        someProp: 'foobar'
      }
    })
  ]
)
```

### Objet de données dans le détail

Une chose est à noter : de la même manière que `v-bind:class` et `v-bind:style` ont un traitement spécial dans les templates, ils ont leurs propres champs dans les objets de données VNode. Cet objet vous permet également d'insérer des attributs HTML normaux ainsi que des propriétés du DOM comme `innerHTML` (cela remplace la directive `v-html`) :

``` js
{
  // Même API que `v-bind:class`, acceptant autant
  // une chaine de caractères, un objet, ou un tableau d'objects.
  class: {
    foo: true,
    bar: false
  },
  // Même API que `v-bind:style`, acceptant autant
  // une chaine de caractères, un objet, ou un tableau d'objects.
  style: {
    color: 'red',
    fontSize: '14px'
  },
  // Attributs HTML normaux
  attrs: {
    id: 'foo'
  },
  // Props des composants
  props: {
    myProp: 'bar'
  },
  // Propriétés du DOM
  domProps: {
    innerHTML: 'baz'
  },
  // La gestion d'évènements est regroupée sous `on` cependant
  // les modificateurs comme `v-on:keyup.enter` ne sont pas
  // supportés. Vous devez vérifier manuellement le code de touche
  // dans le gestionnaire à la place.
  on: {
    click: this.clickHandler
  },
  // Pour les composants seulement. Vous permet d'écouter les
  // évènements natifs, plutôt que ceux émis par
  // le composant via `vm.$emit`.
  nativeOn: {
    click: this.nativeClickHandler
  },
  // Directives personnalisées. Notez que la valeur `oldValue`
  // de la liaison ne peut pas être affectée, car Vue la conserve
  // pour vous.
  directives: [
    {
      name: 'my-custom-directive',
      value: '2',
      expression: '1 + 1',
      arg: 'foo',
      modifiers: {
        bar: true
      }
    }
  ],
  // Slots internes sous la forme
  // { name: props => VNode | Array<VNode> }
  scopedSlots: {
    default: props => createElement('span', props.text)
  },
  // Le nom du slot, si ce composant est
  // l'enfant d'un autre composant
  slot: 'name-of-slot',
  // Autres propriétés spéciales de premier niveau
  key: 'myKey',
  ref: 'myRef',
  // Si vous appliquez le même nom de `ref` à plusieurs
  // elements dans la fonction de rendu. Cela va faire de `$refs.myRef` un
  // tableau
  refInFor: true
}
```

### Exemple complet

Avec toutes ces informations, nous pouvons finir le composant que nous avons commencé :

``` js
var getChildrenTextContent = function (children) {
  return children.map(function (node) {
    return node.children
      ? getChildrenTextContent(node.children)
      : node.text
  }).join('')
}

Vue.component('anchored-heading', {
  render: function (createElement) {
    // créer un identifiant avec la kebab-case
    var headingId = getChildrenTextContent(this.$slots.default)
      .toLowerCase()
      .replace(/\W+/g, '-')
      .replace(/(^-|-$)/g, '')

    return createElement(
      'h' + this.level,
      [
        createElement('a', {
          attrs: {
            name: headingId,
            href: '#' + headingId
          }
        }, this.$slots.default)
      ]
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

### Contraintes

#### Les VNodes doivent être uniques

Tous les VNodes dans l'arbre des composants doivent être uniques. Cela signifie que la fonction de rendu suivante est invalide :

``` js
render: function (createElement) {
  var myParagraphVNode = createElement('p', 'hi')
  return createElement('div', [
    // Aïe - VNodes dupliqués !
    myParagraphVNode, myParagraphVNode
  ])
}
```

Si vous souhaitez réellement dupliquer le même élément/composant plusieurs fois, vous pouvez le faire avec une fonction fabrique. Par exemple, la fonction de rendu suivante est parfaitement valide pour faire le rendu de 20 paragraphes identiques :

``` js
render: function (createElement) {
  return createElement('div',
    Array.apply(null, { length: 20 }).map(function () {
      return createElement('p', 'hi')
    })
  )
}
```

## Remplacer les fonctionnalités de template en pur JavaScript

### `v-if` et `v-for`

Partout où quelque chose peut être accompli simplement en JavaScript, les fonctions de rendu de Vue ne fournissent pas d'alternative propriétaire. Par exemple, un template utilisant `v-if` et `v-for` :

``` html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>Aucun élément trouvé.</p>
```

Cela pourrait être réécrit avec les `if`/`else` et `map` du JavaScript dans une fonction de rendu

``` js
props: ['items'],
render: function (createElement) {
  if (this.items.length) {
    return createElement('ul', this.items.map(function (item) {
      return createElement('li', item.name)
    }))
  } else {
    return createElement('p', 'Aucun élément trouvé.')
  }
}
```

### `v-model`

Il n'y a pas d'équivalent à `v-model` dans les fonctions de rendu. Vous devez implémenter la logique vous-même :

``` js
props: ['value'],
render: function (createElement) {
  var self = this
  return createElement('input', {
    domProps: {
      value: self.value
    },
    on: {
      input: function (event) {
        self.$emit('input', event.target.value)
      }
    }
  })
}
```

C'est le prix à payer pour travailler au plus bas niveau, mais cela vous donne un meilleur contrôle sur le détail des interactions comparé à `v-model`.

### Modificateurs d'évènement et de code de touche

Pour les modificateurs d'évènement `.passive`, `.capture` et `.once`, Vue offre des préfixes pouvant être utilisés dans `on`:

| Modificateur(s) | Préfixes |
| ------ | ------ |
| `.passive` | `&` |
| `.capture` | `!` |
| `.once` | `~` |
| `.capture.once` ou<br>`.once.capture` | `~!` |

Par exemple :

```javascript
on: {
  '!click': this.doThisInCapturingMode,
  '~keyup': this.doThisOnce,
  '~!mouseover': this.doThisOnceInCapturingMode
}
```

Pour tous les autres modificateurs d'évènement et de code de touche, aucun préfixe propriétaire n'est nécessaire car il suffit d'utiliser des méthodes d'évènement dans le gestionnaire :

| Modificateur(s) | Équivalence dans le gestionnaire |
| ------ | ------ |
| `.stop` | `event.stopPropagation()` |
| `.prevent` | `event.preventDefault()` |
| `.self` | `if (event.target !== event.currentTarget) return` |
| Touches :<br>`.enter`, `.13` | `if (event.keyCode !== 13) return` (changez `13` en un [autre code de touche](http://keycode.info/) pour les autres modificateurs de code de touche) |
| Modificateurs de Clés :<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return` (changez respectivement `ctrlKey` par `altKey`, `shiftKey`, ou `metaKey`) |

Voici un exemple avec tous ces modificateurs utilisés ensemble :

```javascript
on: {
  keyup: function (event) {
    // Annuler si l'élément qui émet l'évènement n'est pas
    // l'élément auquel l'évènement est lié
    if (event.target !== event.currentTarget) return
    // Annuler si la touche relâchée n'est pas la touche
    // Entrée (13) et si la touche `shift` n'est pas maintenue
    // en même temps
    if (!event.shiftKey || event.keyCode !== 13) return
    // Arrêter la propagation d'évènement
    event.stopPropagation()
    // Éviter la gestion par défaut de cet évènement pour cet élément
    event.preventDefault()
    // ...
  }
}
```

### Slots

Vous pouvez accéder aux contenus des slots statiques en tant que tableaux de VNodes depuis [`this.$slots`](../api/#vm-slots) :

``` js
render: function (createElement) {
  // `<div><slot></slot></div>`
  return createElement('div', this.$slots.default)
}
```

Et accéder aux slots de portée en tant que fonctions qui retournent des VNodes via [`this.$scopedSlots`](../api/#vm-scopedSlots) :

``` js
props: ['message'],
render: function (createElement) {
  // `<div><slot :text="message"></slot></div>`
  return createElement('div', [
    this.$scopedSlots.default({
      text: this.message
    })
  ])
}
```

Pour passer des slots internes à un composant enfant en utilisant des fonctions de rendu, utilisez la propriété `scopedSlots` dans les données du VNode :

``` js
render: function (createElement) {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return createElement('div', [
    createElement('child', {
      // passer `scopedSlots` dans l'objet de données
      // in the form of { name: props => VNode | Array<VNode> }
      scopedSlots: {
        default: function (props) {
          return createElement('span', props.text)
        }
      }
    })
  ])
}
```

## JSX

Si vous écrivez beaucoup de fonctions `render`, cela pourra vous sembler fatiguant d'écrire des choses comme ça :

``` js
createElement(
  'anchored-heading', {
    props: {
      level: 1
    }
  }, [
    createElement('span', 'Hello'),
    ' world!'
  ]
)
```

Et d'autant plus quand la version template est vraiment simple en comparaison :

``` html
<anchored-heading :level="1">
  <span>Hello</span> world !
</anchored-heading>
```

C'est pourquoi il y a un [plugin Babel](https://github.com/vuejs/jsx) pour utiliser JSX avec Vue, nous permettant l'utilisation d'une syntaxe plus proche de celle des templates :

``` js
import AnchoredHeading from './AnchoredHeading.vue'

new Vue({
  el: '#demo',
  render: function (h) {
    return (
      <AnchoredHeading level={1}>
        <span>Hello</span> world !
      </AnchoredHeading>
    )
  }
})
```

<p class="tip">Utiliser `h` comme alias de `createElement` est une convention courante que vous verrez dans l'écosystème Vue et qui est en faite requise pour JSX. À partir de la [version 3.4.0](https://github.com/vuejs/babel-plugin-transform-vue-jsx#h-auto-injection) du plugin Babel pour Vue, nous injections automatiquement `const h = this.$createElement` dans n'importe quelle méthode ou accesseur (pas dans les fonctions ou fonctions fléchées), déclaré avec la syntaxe ES2015 qui a du JSX, vous pouvez ainsi oublier le paramètre `(h)`. Avec les versions précédentes du plugin, si `h` n'est pas disponible dans votre portée courante, votre application va lever une erreur.</p>

Pour plus d'informations sur comment utiliser JSX dans du JavaScript, référez-vous à la [documentation d'utilisation](https://github.com/vuejs/jsx#installation).

## Composants fonctionnels

Le composant de titre ancré que nous avons créé plus tôt était relativement simple. Il ne gère aucun état, n'observe aucun état qu'on lui passe, et il n'a pas de méthodes de cycle de vie. Non, ce n'est qu'une simple fonction avec quelques props.

Dans des cas comme celui-ci, nous pouvons marquer les composants comme `functional`, ce qui signifie qu'ils sont sans état (« stateless » c.-à-d. sans [`data` réactive](../api/#Options-Data)) et sans instance (« instanceless » c.-à-d. sans contexte `this`). Un **composant fonctionnel** ressemble à ça :

``` js
Vue.component('my-component', {
  functional: true,
  // Les props sont optionnelles
  props: {
    // ...
  },
  // Pour compenser le manque d'instance,
  // nous pouvons maintenant fournir en second argument un contexte.
  render: function (createElement, context) {
    // ...
  }
})
```

> Note : dans les versions avant 2.3.0, l'option `props` est requise si vous souhaitez accepter des props dans un composant fonctionnel. Dans les versions 2.3.0+ vous pouvez omettre l'option `props` et tous les attributs trouvés dans le nœud composant seront implicitement extraits comme des props.
> 
> La référence sera HTMLElement quand elle sera utilisé avec des composants fonctionnels car elles sont sans état et sans instance.

Dans la 2.5.0+, si vous utilisez les [composants monofichiers](single-file-components.html), les templates fonctionnels basés sur les composants peuvent être déclarés avec :

``` html
<template functional>
</template>
```

Tout ce dont le composant a besoin est passé dans l'objet `context`, qui est un objet contenant :

- `props` : un objet avec les props fournies,
- `children` : un tableau de VNode enfants,
- `slots` : une fonction retournant un objet de slots,
- `scopedSlots`: (2.6.0+) un objet qui expose les portées passées dans les slots. Expose également les slots normaux comme des fonctions,
- `data` : l'[objet de données](#Objet-de-donnees-dans-le-detail) (`data`) complet passé au composant en tant que second argument de `createElement`,
- `parent` : une référence au composant parent,
- `listeners` : (2.3.0+) un objet contenant les écouteurs d'évènement enregistrés dans le parent. C'est un simple alias de `data.on`,
- `injections` : (2.3.0+) si vous utilisez l'option [`inject`](../api/#provide-inject), cela va contenir les injections résolues.

Après avoir ajouté `functional: true`, mettre à jour la fonction de rendu de notre composant de titres avec ancres va simplement nécessiter d'ajouter l'argument `context`, en remplaçant `this.$slots.default` par `context.children`, puis `this.level` par `context.props.level`.

Puisque les composants fonctionnels ne sont que des fonctions, leur rendu est plus rapide.

Ils sont également très utiles en tant que composants enveloppes. Par exemple, quand vous avez besoin de :

- Programmatiquement choisir un composant parmi plusieurs autres composants pour de la délégation ou
- Manipuler les enfants, props, ou données avant de les passer au composant enfant.

Voici un exemple d'un composant `smart-list` qui délègue à des composants plus spécifiques en fonction des props qui lui sont passées :

``` js
var EmptyList = { /* ... */ }
var TableList = { /* ... */ }
var OrderedList = { /* ... */ }
var UnorderedList = { /* ... */ }

Vue.component('smart-list', {
  functional: true,
  props: {
    items: {
      type: Array,
      required: true
    },
    isOrdered: Boolean
  },
  render: function (createElement, context) {
    function appropriateListComponent () {
      var items = context.props.items

      if (items.length === 0)           return EmptyList
      if (typeof items[0] === 'object') return TableList
      if (context.props.isOrdered)      return OrderedList

      return UnorderedList
    }

    return createElement(
      appropriateListComponent(),
      context.data,
      context.children
    )
  }
})
```

### Passer des attributs et évènements aux éléments / composants enfants

Sur les composants normaux, les attributs qui ne sont pas définis comme props sont automatiquement ajoutés à l'élément racine du composant en remplaçant ou [en étant intelligemment rajouté sur](class-and-style.html) n'importe quel attribut existant avec le même nom.

Cependant les composants fonctionnels vous demande de définir explicitement ce comportement :

```js
Vue.component('my-functional-button', {
  functional: true,
  render: function (createElement, context) {
    // Passer de manière transparente n'importe quel attribut, écouteur d'évènement, enfant, etc.
    return createElement('button', context.data, context.children)
  }
})
```

En passant `context.data` en tant que second paramètre à `createElement`, nous transferrons à l'enfant racine n'importe quel attribut ou écouteur d'évènement utilisé sur `my-functional-button`. C'est même tellement transparent que les évènements n'ont même pas besoin du modificateur `.native`.

Si vous utilisez un composant fonctionnel basé sur un template, vous devrez aussi ajouter manuellement les attributs et les écouteurs. Puisque nous avons accès au contenus individuels du contexte, nous pouvons utiliser `data.attrs` en le passant avec n'importes quel attribut HTML ou `listeners` _(l'alias pour `data.on`)_ en le passant avec n'importe quel écouteur d'évènement.

```html
<template functional>
  <button
    class="btn btn-primary"
    v-bind="data.attrs"
    v-on="listeners"
  >
    <slot/>
  </button>
</template>
```

### `slots()` vs. `children`

Vous vous demandez peut-être l'utilité d'avoir `slot()` et `children` en même temps. `slots().default` n'est-il pas la même chose que `children` ? Dans la majorité des cas, oui. Mais que faire si vous avez un composant fonctionnel avec les enfants suivants ?

``` html
<my-functional-component>
  <p v-slot:foo>
    premier
  </p>
  <p>second</p>
</my-functional-component>
```

Pour ce composant, `children` va vous donner les deux paragraphes, `slots().default` ne vous donnera que le second, et `slots().foo` ne vous donnera que le premier. Avoir le choix entre `children` et `slots()` vous permet donc de choisir ce que le composant sait à propos du système de slot ou si vous déléguez peut-être la responsabilité à un autre composant en passant simplement `children`.

## Compilation de template

Vous serez peut-être intéressé de savoir que les templates Vue sont en fait compilés en fonctions de rendu. C'est un détail d'implémentation dont vous n'avez en général pas à vous soucier, mais si vous souhaitez savoir comment un template spécifique est compilé, vous pourrez trouver cela intéressant. Vous trouverez ci-dessous une petite démo utilisant `Vue.compile` pour voir en live le rendu d'une chaine de template :

<iframe src="https://codesandbox.io/embed/github/vuejs/v2.vuejs.org/tree/master/src/v2/examples/vue-20-template-compilation?codemirror=1&hidedevtools=1&hidenavigation=1&theme=light&view=preview" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" title="vue-20-template-compilation" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>
