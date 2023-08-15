<script setup>
import { onMounted, ref } from "vue";
import { Command } from "@tauri-apps/api/shell";
import { listen } from "@tauri-apps/api/event";
import { open, message } from "@tauri-apps/api/dialog";

const patching = ref(false);

listen("tauri://file-drop", (event) => {
  console.log(event);
});

async function checkNvPatchInstalled() {
  // execute powershell code and get output using tauri API
  const { stdout } = await new Command(
    "powershell",
    '-command dotnet tool list -g | Select-String -Pattern "nvpatch" -Quiet'.split(
      " "
    )
  ).execute();
  return stdout.trim() === "True";
}

async function maybeInstallNvPatch() {
  if (!(await checkNvPatchInstalled())) {
    await new Command("powershell", [
      "-command",
      "dotnet",
      "tool",
      "install",
      "-g",
      "Topten.nvpatch",
    ]).execute();
  }
}

async function nvPatch(file) {
  const { stdout } = await new Command("powershell", [
    "-command",
    "nvpatch",
    "--enable",
    `"${file}"`,
  ]).execute();

  return stdout.trim() === "OK";
}

async function handleFile(file) {
  patching.value = true;
  if (file === null) return;
  if (file instanceof Array) {
    if (file.length === 0) return;
    file = file[0];
  }
  if (file.split(".").pop() !== "exe") return;

  await maybeInstallNvPatch();
  const success = await nvPatch(file);
  if (success) {
    message(`Successfully patched ${file}!`);
  } else {
    message(`Failed to patch ${file}!`);
  }
  patching.value = false;
}

onMounted(() => {
  // drag and drop zone logic
  const dropzone = document.querySelector(".dropzone");

  dropzone.addEventListener("dragover", (e) => {
    e.preventDefault();
    dropzone.classList.add("dropzone--active");
  });

  dropzone.addEventListener("dragleave", (e) => {
    e.preventDefault();
    dropzone.classList.remove("dropzone--active");
  });

  listen("tauri://file-drop", (event) => {
    console.log(event);
    handleFile(event.payload);
  });

  dropzone.addEventListener("drop", (e) => {
    e.preventDefault();
    dropzone.classList.remove("dropzone--active");
    // handleFile(file);
  });

  dropzone.addEventListener("click", () => {
    open({
      multiple: false,
      filters: [{ name: "Executable", extensions: ["exe"] }],
    })
      .then((result) => {
        if (result !== undefined) {
          handleFile(result);
        }
      })
      .catch((err) => {
        console.log(err);
      });
  });
});
</script>

<template>
  <div class="container">
    <h1>NVPatch UI</h1>
    <!-- drag and drop or select file UI -->
    <div class="dropzone" v-if="!patching">
      <div class="dropzone__prompt">Drop your .exe here or click to select</div>
    </div>
    <!-- patching UI -->
    <div v-else>Patching...</div>
  </div>
</template>

<style scoped>
.logo.vite:hover {
  filter: drop-shadow(0 0 2em #747bff);
}

.logo.vue:hover {
  filter: drop-shadow(0 0 2em #249b73);
}

.dropzone {
  width: calc(100% - 4rem - 9px);
  height: calc(78vh - 4rem - 2px);
  border: 2px dashed #747bff;
  border-radius: 5px;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-left: 3px;
  user-select: none;
}

h1 {
  user-select: none;
}

.dropzone--active {
  border-color: #249b73;
}

.dropzone__prompt {
  font-size: 1.5rem;
  font-weight: 500;
  color: #747bff;
}

.dropzone__input {
  display: none;
}

html {
  height: 100%;
  width: 100%;
  padding: 0;
  margin: 0;
  overflow: none;
}

body {
  height: 100%;
  width: 100%;
  padding: 0;
  margin: 0;
  overflow: none;
}

#app {
  height: 100%;
  width: 100%;
  padding: 0;
  margin: 0;
  overflow: none;
  user-select: none;
}
</style>
