<script>
  import { onMount } from "svelte";

  import * as Lib from "../lib";
  import PreviewFrame from "$lib/preview-frame/index.svelte";
  import LoadButton from "$lib/load-button.svelte";

  import NameReplacements from "./name-replacements.svelte";
  import AspectEffects from "./aspect-effects.svelte";
  import CustomIcons from "../custom-icons.svelte";
  import { downloadHTML } from "$lib/download";

  export let aspect;
  export let instructions;
  export let emptyAspect;
  export let customIcons;

  let previewFrame;

  async function loadHTMLFromURL(url) {
    url = new URL(url, document.baseURI);
    let loadedDocument = await Lib.loadHTML(url);
    readHTML(loadedDocument, url);
    reloadPreview();
  }

  const demoURL = "/template/MyCustomContent/MyAspect/aspect_website.html";
  function onLoad() {
    if (aspect.demoBoardWasLoaded === false) {
      loadHTMLFromURL(demoURL).then(() => {
        aspect.demoBoardWasLoaded = true;
        emptyAspect.demoBoardWasLoaded = true;
      });
    } else {
      reloadPreview();
    }
  }
  onMount(onLoad);

  function reloadPreview() {
    console.log("Updating Preview (f=reloadPreview)");
    previewFrame.copyHTMLFrom(generateHTML(aspect)).then(() => {
      previewFrame.startMain();
    });
  }

  function generateHTML(aspect) {
    const fragment = new DocumentFragment();

    let aspectHTML = document.createElement("aspect");
    fragment.append(aspectHTML);

    //Set Aspect Name
    let aspectName = document.createElement("aspect-name");
    aspectName.innerHTML = aspect.nameReplacements.aspectName;
    aspectHTML.appendChild(aspectName);

    //Profile or Landscape
    if (aspect.profile) {
      aspectHTML.setAttribute("profile", "");
    }

    //Set Replacement
    aspect.nameReplacements.replacements.forEach((replacement) => {
      let aspectReplacementHTML = document.createElement("aspect-subtext");
      aspectHTML.appendChild(aspectReplacementHTML);
      let replacementFullText = replacement.aspectRelacement;
      if (replacement.rulesReplaced) {
        replacementFullText += ": <i>" + replacement.rulesReplaced + "</i>";
      }
      if (aspectReplacementHTML) {
        aspectReplacementHTML.innerHTML = replacementFullText;
      }
    });

    //Set Complexity
    if (aspect.nameReplacements.complexity) {
      const complexityHTML = document.createElement("complexity");
      aspectHTML.appendChild(complexityHTML);
      if (complexityHTML) {
        complexityHTML.setAttribute("value", aspect.nameReplacements.complexity);
      } else {
        const newComplexityHTML = document.createElement("complexity");
        newComplexityHTML.setAttribute("value", aspect.nameReplacements.complexity);
        aspectHTML.appendChild(newComplexityHTML);
      }
    }

    //Set Aspect Back
    if (aspect.nameReplacements.hasBack) {
      let aspectBackHTML = document.createElement("aspect-back");
      fragment.append(aspectBackHTML);
      aspectBackHTML.setAttribute("spirit-name", aspect.nameReplacements.spiritName);
      aspectBackHTML.setAttribute("src", aspect.nameReplacements.spiritImage);
    }

    //Set Special Rules
    const aspectRulesContainer = document.createElement("aspect-container");
    aspectHTML.appendChild(aspectRulesContainer);

    aspect.aspectEffects.specialRules.rules.forEach((rule) => {
      let newRuleName = document.createElement("special-rules-subtitle");
      newRuleName.textContent = rule.name;
      let newRuleEffect = document.createElement("special-rule");
      newRuleEffect.innerHTML = rule.effect;
      aspectRulesContainer.appendChild(newRuleName);
      aspectRulesContainer.appendChild(newRuleEffect);
    });

    //Set Innate Powers
    aspect.aspectEffects.innatePowers.powers.forEach((power) => {
      let newInnatePower = document.createElement("quick-innate-power");
      newInnatePower.setAttribute("name", power.name);
      newInnatePower.setAttribute("speed", power.speed.toLowerCase());
      newInnatePower.setAttribute("range", power.range);
      newInnatePower.setAttribute("target", power.target);
      newInnatePower.setAttribute("target-title", power.targetTitle);
      if (power.note) {
        newInnatePower.setAttribute("note", power.note);
      } // may need to clear it?
      power.levels.forEach((level) => {
        let newLevel = document.createElement("level");
        newLevel.setAttribute("threshold", level.threshold);
        newLevel.textContent = level.effect;
        if (level.isLong) {
          newLevel.setAttribute("long", "");
        }
        newInnatePower.appendChild(newLevel);
      });
      aspectRulesContainer.appendChild(newInnatePower);
    });

    //Set Custom Icons
    const spiritStyle = document.createElement("style");
    fragment.prepend(spiritStyle);
    let customIconText = "";
    customIcons.icons.forEach((icon) => {
      customIconText +=
        "icon.custom" + (icon.id + 1) + "{background-image: url('" + icon.name + "'); }\n";
    });
    spiritStyle.textContent = customIconText;

    return fragment;
  }

  function readHTML(htmlElement, baseURI) {
    console.log("Loading aspect into form (f=readHTML)");
    //Reads the Template HTML file into the Form
    const aspectHTML = htmlElement.querySelectorAll("aspect")[0];

    //Profile or Landscape
    if (aspectHTML.hasAttribute("profile")) {
      aspect.profile = true;
    }

    //Read Aspect Name
    const aspectName = aspectHTML.querySelectorAll("aspect-name")[0];
    aspect.nameReplacements.aspectName = aspectName.innerHTML.trim();

    //Read Replacement(s)
    aspect.nameReplacements.replacements.splice(0, aspect.nameReplacements.replacements.length); //Clear the Form first
    const aspectReplacementsHTML = aspectHTML.querySelectorAll("aspect-subtext");
    if (aspectReplacementsHTML) {
      aspectReplacementsHTML.forEach((replacement) => {
        aspect.nameReplacements.replacements.push({
          id: aspect.nameReplacements.replacements.length,
          aspectRelacement: replacement.textContent.split(":")[0],
          rulesReplaced: replacement.querySelectorAll("i")[0].textContent,
        });
        aspect = aspect;
      });
    }

    //Read Complexity
    const complexityHTML = aspectHTML.querySelectorAll("complexity")[0];
    if (complexityHTML) {
      aspect.nameReplacements.complexity = complexityHTML.getAttribute("value");
    }

    //Read Aspect Back
    const aspectBackHTML = htmlElement.querySelectorAll("aspect-back")[0];
    if (aspectBackHTML) {
      aspect.nameReplacements.spiritName = aspectBackHTML.getAttribute("spirit-name");
      aspect.nameReplacements.spiritImage = Lib.maybeResolveURL(
        aspectBackHTML.getAttribute("src"),
        baseURI
      );
      aspect.nameReplacements.hasBack = true;
    } else {
      aspect.nameReplacements.hasBack = false;
    }

    //Read Special Rules
    const specialRulesNames = aspectHTML.querySelectorAll("special-rules-subtitle");
    const specialRulesEffects = aspectHTML.querySelectorAll("special-rule");
    aspect.aspectEffects.specialRules.rules.splice(
      0,
      aspect.aspectEffects.specialRules.rules.length
    ); //Clear the Form first
    specialRulesNames.forEach((specialRulesName, j) => {
      aspect.aspectEffects = Lib.addSpecialRule(
        aspect.aspectEffects,
        specialRulesName.textContent,
        specialRulesEffects[j].innerHTML.trim()
      );
      aspect = aspect;
    });

    //Read Innate Powers
    let innatePowers = htmlElement.querySelectorAll("quick-innate-power");
    aspect.aspectEffects.innatePowers.powers.splice(
      0,
      aspect.aspectEffects.innatePowers.powers.length
    ); //Clear the Form first
    if (innatePowers) {
      innatePowers.forEach((innatePower, k) => {
        aspect.aspectEffects = Lib.addInnatePower(
          aspect.aspectEffects,
          innatePower.getAttribute("name"),
          innatePower.getAttribute("speed"),
          innatePower.getAttribute("range"),
          innatePower.getAttribute("target"),
          innatePower.getAttribute("target-title"),
          innatePower.getAttribute("note")
        );
        let htmlLevels = innatePower.querySelectorAll("level");
        htmlLevels.forEach((htmlLevel) => {
          aspect.aspectEffects = Lib.addLevel(
            aspect.aspectEffects,
            k,
            htmlLevel.getAttribute("threshold"),
            htmlLevel.textContent.trim(),
            htmlLevel.hasAttribute("long")
          );
        });
      });
    }

    //Custom Icons
    if (aspect.demoBoardWasLoaded) {
      const aspectStyle = htmlElement.querySelectorAll("style")[0];
      customIcons.icons.splice(0, customIcons.icons.length); //Clear the Form first
      if (aspectStyle) {
        const regExp = new RegExp(/(?<=(["']))(?:(?=(\\?))\2.)*?(?=\1)/, "g");
        let iconList = aspectStyle.textContent.match(regExp);
        if (iconList) {
          iconList.forEach((customIcon) => {
            customIcons = Lib.addCustomIcon(customIcons, customIcon);
            console.log(customIcon);
          });
        }
      }
    } else {
      console.log("SKIPPING ICON LOAD");
    }

    console.log("aspect loaded");
    console.log(aspect);
  }

  function exportAspect() {
    const htmlFileName = aspect.nameReplacements.aspectName.replaceAll(" ", "_") + "_Aspect.html";
    downloadHTML(generateHTML(aspect), htmlFileName);
  }

  function clearAllFields() {
    if (window.confirm("Are you sure? This permanently clears all fields in Aspect.")) {
      aspect = JSON.parse(JSON.stringify(emptyAspect));
      reloadPreview();
    }
  }

  function showInstructions() {
    instructions.open("power-cards");
  }

  function screenshotSetUp() {
    const fileNames = [
      aspect.nameReplacements.aspectName.replaceAll(" ", "_") + "_Aspect.png",
      aspect.nameReplacements.aspectName.replaceAll(" ", "_") + "_AspectBack.png",
    ];
    const elementNamesInIframe = ["aspect", "aspect-back"];
    previewFrame.takeScreenshot(fileNames, elementNamesInIframe);
  }
</script>

<PreviewFrame id="aspect-preview" bind:this={previewFrame} on:hot-reload={reloadPreview}>
  <svelte:fragment slot="head">
    <link href="/template/_global/css/global.css" rel="stylesheet" />
    <link href="/template/_global/css/aspect.css" rel="stylesheet" />
    <script type="text/javascript" src="/template/_global/js/common.js"></script>
    <script type="text/javascript" src="/template/_global/js/aspect.js"></script>
  </svelte:fragment>
</PreviewFrame>

<div class="field has-addons mb-2">
  <LoadButton accept=".html" class="button is-success mr-1" loadObjectURL={loadHTMLFromURL}>
    Load
  </LoadButton>
  <button class="button is-success  mr-1" on:click={exportAspect}> Save </button>
  <button class="button is-success  mr-1" on:click={screenshotSetUp}>Download Image</button>
  <button class="button is-warning  mr-1" on:click={reloadPreview}>Update Preview</button>
  <button class="button is-warning mr-1" on:click={previewFrame.toggleSize}
    >Toggle Board Size</button>
  <button class="button is-danger mr-1" on:click={clearAllFields}>Clear All Fields</button>
  <button class="button is-info  mr-1" on:click={showInstructions}>Instructions</button>
</div>
<div class="columns mt-0 mb-1">
  <div class="column pt-0">
    <NameReplacements bind:aspect />
    <CustomIcons bind:customIcons />
    <!-- <CustomIcons bind:customIcons /> -->
  </div>
  <div class="column pt-0">
    <AspectEffects bind:aspect />
  </div>
</div>
