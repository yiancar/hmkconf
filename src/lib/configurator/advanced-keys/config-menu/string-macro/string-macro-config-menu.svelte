<!--
This program is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later
version.

This program is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
FOR A PARTICULAR PURPOSE. See the GNU General Public License for more
details.

You should have received a copy of the GNU General Public License along with
this program. If not, see <https://www.gnu.org/licenses/>.
-->

<script lang="ts">
  import { PlusIcon, TrashIcon } from "@lucide/svelte"
  import FixedScrollArea from "$lib/components/fixed-scroll-area.svelte"
  import KeycodeAccordion from "$lib/components/keycode-accordion.svelte"
  import { KeycodeButton } from "$lib/components/keycode-button"
  import { Button } from "$lib/components/ui/button"
  import { Input } from "$lib/components/ui/input"
  import { advancedKeysQueryContext } from "$lib/configurator/queries/advanced-keys-query.svelte"
  import { stringMacrosQueryContext } from "$lib/configurator/queries/string-macros-query.svelte"
  import { keyboardContext } from "$lib/keyboard"
  import {
    HMK_AKType,
    HMK_StringMacroAction,
    type HMK_AKStringMacro,
    type HMK_StringMacroStep,
  } from "$lib/libhmk/advanced-keys"
  import { Keycode } from "$lib/libhmk/keycodes"
  import { unitToStyle } from "$lib/ui"
  import { ToggleGroup } from "bits-ui"
  import { configMenuStateContext } from "../context.svelte"

  const configMenuState = configMenuStateContext.get()
  const advancedKeysQuery = advancedKeysQueryContext.get()
  const stringMacrosQuery = stringMacrosQueryContext.get()
  const keyboard = keyboardContext.get()
  const { stringMacroBufferSize, stringMacroDelayUnitMs } = keyboard.metadata

  const action = $derived(
    configMenuState.advancedKey.action as HMK_AKStringMacro,
  )
  const advancedKeys = $derived(advancedKeysQuery.advancedKeys.current)
  const stringMacros = $derived(stringMacrosQuery.stringMacros.current)
  let selectedStep = $state<string>("")
  let loadedKey = $state("")
  let draftSteps = $state<HMK_StringMacroStep[]>([])

  function decodeSteps() {
    if (!stringMacros || action.len % 3 !== 0) return []

    const ret: HMK_StringMacroStep[] = []
    for (let i = 0; i < action.len; i += 3) {
      ret.push({
        keycode: stringMacros[action.offset + i] ?? Keycode.KC_NO,
        action:
          stringMacros[action.offset + i + 1] ?? HMK_StringMacroAction.TAP,
        delay: stringMacros[action.offset + i + 2] ?? 1,
      })
    }
    return ret
  }

  function encodeSteps(steps: HMK_StringMacroStep[]) {
    return steps.flatMap(({ keycode, action, delay }) => [
      keycode,
      action,
      delay,
    ])
  }

  const savedBytes = $derived(encodeSteps(decodeSteps()))
  const draftBytes = $derived(encodeSteps(draftSteps))
  const draftOffset = $derived(allocateMacro(draftBytes))
  const hasStorageError = $derived(draftOffset === null)
  const hasUnsavedChanges = $derived(
    draftBytes.length !== savedBytes.length ||
      draftBytes.some((byte, i) => byte !== savedBytes[i]),
  )

  $effect(() => {
    if (!stringMacros) return

    const key = `${action.offset}:${action.len}:${savedBytes.join(",")}`
    if (key === loadedKey) return

    draftSteps = decodeSteps()
    selectedStep = ""
    loadedKey = key
  })

  function allocateMacro(bytes: number[]) {
    if (!advancedKeys) return null
    if (
      action.offset + bytes.length <= stringMacroBufferSize &&
      bytes.length <= action.len
    ) {
      return action.offset
    }

    const used = Array(stringMacroBufferSize).fill(false)
    for (const advancedKey of advancedKeys) {
      const otherAction = advancedKey.action
      if (
        otherAction.type !== HMK_AKType.STRING_MACRO ||
        otherAction === action
      )
        continue
      for (let i = 0; i < otherAction.len; i++) {
        if (otherAction.offset + i < used.length) {
          used[otherAction.offset + i] = true
        }
      }
    }

    for (
      let offset = 0;
      offset <= stringMacroBufferSize - bytes.length;
      offset++
    ) {
      if (used.slice(offset, offset + bytes.length).every((v) => !v)) {
        return offset
      }
    }
    return null
  }

  async function commitDraftSteps() {
    const bytes = encodeSteps(draftSteps)
    const offset = draftOffset
    if (offset === null) return

    if (bytes.length > 0) {
      await stringMacrosQuery.set({ offset, data: bytes })
    }
    await configMenuState.updateAction({
      type: HMK_AKType.STRING_MACRO,
      offset,
      len: bytes.length,
    })
  }

  $effect(() => {
    configMenuState.setSave(async () => {
      if (hasUnsavedChanges) {
        await commitDraftSteps()
      }
    })

    return () => {
      configMenuState.setSave(null)
      configMenuState.setCanSave(true)
    }
  })

  $effect(() => {
    configMenuState.setCanSave(!hasStorageError)
  })

  function updateStep(index: number, patch: Partial<HMK_StringMacroStep>) {
    draftSteps = draftSteps.map((step, i) =>
      i === index ? { ...step, ...patch } : step,
    )
  }

  function removeStep(index: number) {
    draftSteps = draftSteps.filter((_, i) => i !== index)
  }

  function addStep(keycode = Keycode.KC_NO) {
    draftSteps = [
      ...draftSteps,
      {
        keycode,
        action: HMK_StringMacroAction.TAP,
        delay: 1,
      },
    ]
  }

  function selectKeycode(keycode: number) {
    const index = Number(selectedStep)
    if (selectedStep === "" || !Number.isInteger(index)) {
      addStep(keycode)
    } else {
      updateStep(index, { keycode })
      selectedStep = ""
    }
  }

  function delayUnitsToMs(delay: number) {
    return delay * stringMacroDelayUnitMs
  }

  function delayMsToUnits(delayMs: number) {
    return Math.max(
      0,
      Math.min(255, Math.round(delayMs / stringMacroDelayUnitMs)),
    )
  }
</script>

<FixedScrollArea class="flex flex-col gap-4 p-4 pt-0">
  <div class="grid text-sm">
    <span class="font-medium">Configure String Macro Bindings</span>
    <span class="text-muted-foreground">
      Assign macro steps using the menu on the right. Click a keycode in the
      macro to edit it, then press Done to save. Delay values are in
      milliseconds.
    </span>
  </div>

  {#if hasStorageError}
    <div
      class="rounded-md border border-destructive/40 px-3 py-2 text-sm text-destructive"
    >
      Not enough macro storage in this profile.
    </div>
  {/if}

  {#if draftSteps.length > 0}
    <div
      class="grid grid-cols-[5rem_minmax(0,1fr)_5rem_2.25rem] items-center gap-2 text-sm font-medium"
    >
      <span class="text-center">Key</span>
      <span>Action</span>
      <span class="text-center">Delay (ms)</span>
      <span class="sr-only">Delete</span>
    </div>
  {/if}

  <ToggleGroup.Root
    bind:value={selectedStep}
    class="flex flex-col gap-2"
    type="single"
  >
    {#each draftSteps as step, i (i)}
      <div
        class="grid grid-cols-[5rem_minmax(0,1fr)_5rem_2.25rem] items-center gap-2 rounded-md border p-2"
      >
        <ToggleGroup.Item value={String(i)}>
          {#snippet child({ props })}
            <div class="grid place-items-center p-0.5" style={unitToStyle()}>
              <KeycodeButton
                keycode={step.keycode}
                oncontextmenu={(e) => {
                  e.preventDefault()
                  updateStep(i, { keycode: Keycode.KC_NO })
                }}
                {...props}
              />
            </div>
          {/snippet}
        </ToggleGroup.Item>
        <select
          class="h-9 rounded-md border bg-background px-2 text-sm"
          value={step.action}
          onchange={(e) =>
            updateStep(i, {
              action: Number(e.currentTarget.value),
            })}
        >
          <option value={HMK_StringMacroAction.PRESS}>Press</option>
          <option value={HMK_StringMacroAction.TAP}>Tap</option>
          <option value={HMK_StringMacroAction.RELEASE}>Release</option>
        </select>
        <Input
          max={String(255 * stringMacroDelayUnitMs)}
          min="0"
          step={String(stringMacroDelayUnitMs)}
          type="number"
          value={delayUnitsToMs(step.delay)}
          oninput={(e) =>
            updateStep(i, {
              delay: delayMsToUnits(Number(e.currentTarget.value)),
            })}
        />
        <Button
          onclick={() => {
            if (selectedStep === String(i)) selectedStep = ""
            removeStep(i)
          }}
          size="icon"
          variant="outline"
        >
          <TrashIcon />
          <span class="sr-only">Delete Step</span>
        </Button>
      </div>
    {:else}
      <div
        class="rounded-md border border-dashed p-4 text-sm text-muted-foreground"
      >
        Select a keycode to add the first tap step.
      </div>
    {/each}
  </ToggleGroup.Root>

  <Button onclick={() => addStep()} size="sm" variant="outline">
    <PlusIcon />
    Add Empty Step
  </Button>
</FixedScrollArea>

<FixedScrollArea class="flex flex-col gap-4 p-4 pt-0">
  <KeycodeAccordion onKeycodeSelected={selectKeycode} />
</FixedScrollArea>
