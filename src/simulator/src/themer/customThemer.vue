<template>
  <div>
    <form ref="propertiesContainer">
      <div v-for="(property, key) in customTheme" :key="key" class="property">
        <label :for="String(key)">{{ key }} ({{ property.description }})</label>
        <input type="color" :name="String(key)" v-model="customTheme[key]?.color" class="customColorInput"
          @input="updateTheme" />
      </div>
      <a id="downloadThemeFile" style="display: none"></a>
    </form>

    <button @click="applyTheme">Apply Theme</button>
    <button @click="importTheme">Import Theme</button>
    <button @click="exportTheme">Export Theme</button>

    <input type="file" id="importThemeFile" style="display: none" @change="handleFileImport" />
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive, ref, onMounted } from "vue";
import { dots } from "../canvasApi";
import themeOptions from "./themes";
import { updateThemeForStyle } from "./themer";
import { CreateAbstraction } from "./customThemeAbstraction";

export default defineComponent({
  name: "CustomColorThemes",
  setup() {
    // Reactive state for custom theme
    const customTheme = reactive<{ [key: string]: { color: string; description: string; ref: string[] } }>(CreateAbstraction(themeOptions["Custom Theme"]) as { [key: string]: { color: string; description: string; ref: string[] } });

    // Reference to the form container
    const propertiesContainer = ref<HTMLFormElement | null>(null);

    // Function to update the theme and background
    const updateBG = () => dots(true, false, true);

    // Update theme when color input changes
    const updateTheme = (event: Event) => {
      const target = event.target as HTMLInputElement;
      const key = target.name;
      if (!customTheme[key]) {
        console.error(`Theme property ${key} not found`);
        return;
      }
      (customTheme[key] as { color: string; description: string; ref: string[] }).color = target.value;
      try {
        customTheme[key].ref.forEach((property: string) => {
          themeOptions["Custom Theme"][property] = target.value;
        });
        updateThemeForStyle("Custom Theme");
        updateBG();
      } catch (error) {
        console.error('Failed to update theme:', error);
      }
    };

    // Apply the custom theme
    const applyTheme = () => {
      try {
        if (typeof localStorage === 'undefined') {
          throw new Error('localStorage is not available');
        }
        localStorage.setItem("theme", "Custom Theme");
        localStorage.setItem("Custom Theme", JSON.stringify(themeOptions["Custom Theme"]));
        updateThemeForStyle("Custom Theme");
        updateBG();
      } catch (error) {
        console.error('Failed to save theme:', error);
        // Fallback: Apply theme without saving
        updateThemeForStyle("Custom Theme");
        updateBG();
      }
    };

    // Import the custom theme from a JSON file
    const importTheme = () => {
      const importThemeFile = document.getElementById("importThemeFile") as HTMLInputElement;
      importThemeFile.click();
    };

    // Export the custom theme as a JSON file
    const exportTheme = () => {
      const dlAnchorElem = document.getElementById("downloadThemeFile") as HTMLAnchorElement;
      dlAnchorElem.setAttribute(
        "href",
        `data:text/json;charset=utf-8,${encodeURIComponent(
          JSON.stringify(themeOptions["Custom Theme"])
        )}`
      );
      dlAnchorElem.setAttribute("download", "CV_CustomTheme.json");
      dlAnchorElem.click();
    };

    // Handle file import for custom theme
    const handleFileImport = (event: Event) => {
      const target = event.target as HTMLInputElement;
      const file = target.files?.[0];
      if (!file) {
        alert("No file selected!");
        return;
      }

      if (!file.name.toLowerCase().endsWith('.json')) {
        alert("Please select a JSON file!");
        target.value = "";
        return;
      }

      if (file.size > 1024 * 1024) {  // 1MB limit
        alert("File is too large!");
        target.value = "";
        return;
      }

      const reader = new FileReader();

      reader.onload = (e) => {
        try {
          if (!e.target?.result) {
            throw new Error("Failed to read file");
          }
          const lines = JSON.parse(e.target.result as string);
          if (!lines || typeof lines !== 'object') {
            throw new Error("Invalid theme format");
          }
          Object.assign(customTheme, CreateAbstraction(lines));
          themeOptions["Custom Theme"] = lines;
          updateThemeForStyle("Custom Theme");
          updateBG();
        } catch (error) {
          console.error('Failed to parse theme:', error);
          alert("Invalid theme file!");
        }
      };

      reader.onerror = () => {
        console.error('Failed to read file');
        alert("Failed to read file!");
      };

      reader.readAsText(file);
      target.value = ""; // Reset file input
    };

    // Simulate a delay to update the theme on component mount
    onMounted(() => {
      updateThemeForStyle("Custom Theme");
      updateBG();
    });

    return {
      customTheme,
      propertiesContainer,
      updateTheme,
      applyTheme,
      exportTheme,
      importTheme,
      handleFileImport,
    };
  },
});
</script>

<style scoped>
.property {
  margin-bottom: 10px;
}

.customColorInput {
  margin-left: 10px;
}
</style>