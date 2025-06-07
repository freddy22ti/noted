<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { GradientButton, Modal } from "flowbite-svelte";
  import { Heading, Button } from "flowbite-svelte";
  import { Input, Label, Textarea } from "flowbite-svelte";
  import { Accordion, AccordionItem } from "flowbite-svelte";
  import {
    BanOutline,
    PenSolid,
    CirclePlusOutline,
  } from "flowbite-svelte-icons";
  import { slide } from "svelte/transition";
  import Database from "@tauri-apps/plugin-sql";
  import { onMount } from "svelte";

  interface Note {
    id: number;
    title: string;
    content: string;
    created_at: string;
    updated_at: string;
  }

  let db: Database | null = null;
  let notes = $state<Note[]>([]);
  let add_note_modal = $state(false);
  let update_note_modal = $state(false);

  // svelte-ignore non_reactive_update
  let title = "";
  // svelte-ignore non_reactive_update
  let content = "";
  let currentNoteId: number | null = null;

  onMount(async () => {
    try {
      db = await Database.load("sqlite:noted.db");
      await loadNotes();
    } catch (error) {
      console.error("Failed to initialize database:", error);
    }
  });

  const loadNotes = async () => {
    if (!db) return;
    try {
      const result = await db.select<Note[]>(
        `SELECT * FROM notes ORDER BY created_at DESC`,
      );
      notes = result || [];
    } catch (error) {
      console.error("Failed to load notes:", error);
    }
  };

  const addNote = async () => {
    if (!db || !title || !content) return;
    try {
      await db.execute(
        `INSERT INTO notes (title, content, created_at, updated_at) VALUES (?, ?, ?, ?)`,
        [title, content, new Date().toISOString(), new Date().toISOString()],
      );
      add_note_modal = false;
      await loadNotes();
      title = "";
      content = "";
    } catch (error) {
      console.error("Failed to add note:", error);
    }
  };

  const prepareUpdateNote = (note: Note) => {
    title = note.title;
    content = note.content;
    currentNoteId = note.id;
    update_note_modal = true;
  };

  const updateNote = async () => {
    if (!db || !currentNoteId || !title || !content) return;
    try {
      await db.execute(
        `UPDATE notes SET title = ?, content = ?, updated_at = ? WHERE id = ?`,
        [title, content, new Date().toISOString(), currentNoteId],
      );
      update_note_modal = false;
      await loadNotes();
      title = "";
      content = "";
      currentNoteId = null;
    } catch (error) {
      console.error("Failed to update note:", error);
    }
  };

  const deleteNote = async (id: number) => {
    if (!db) return;
    try {
      await db.execute(`DELETE FROM notes WHERE id = ?`, [id]);
      await loadNotes();
    } catch (error) {
      console.error("Failed to delete note:", error);
    }
  };
</script>

<main class="container p-5">
  <div class="flex justify-between items-center mb-5">
    <Heading tag="h1" class="text-2xl font-bold">Noted</Heading>
    <Button
      outline={false}
      size="md"
      pill={true}
      color="orange"
      class="p-2"
      onclick={() => {
        add_note_modal = true;
      }}
    >
      <CirclePlusOutline class="h-7 w-7" />
    </Button>
  </div>

  <Accordion
    activeClass="bg-blue-300 focus:ring-4 focus:ring-blue-200 text-black "
    inactiveClass="bg-white"
  >
    {#each notes as note (note.id)}
      <AccordionItem>
        {#snippet header()}
          {note.title}
        {/snippet}
        <div class="mb-2 text-black">
          {note.content}
        </div>
        <p class="text-sm text-gray-600">
          Created: {new Date(note.created_at).toLocaleString()}
        </p>
        <p class="text-sm text-gray-600">
          Updated: {new Date(note.updated_at).toLocaleString()}
        </p>

        <div class="flex mt-4">
          <Button
            onclick={() => prepareUpdateNote(note)}
            color="yellow"
            class="ml-auto p-2!"
            size="sm"
          >
            <PenSolid class="h-7 w-7" color="white" />
          </Button>
          <Button
            onclick={() => deleteNote(note.id)}
            color="red"
            class="ml-2 p-2!"
            size="sm"
          >
            <BanOutline class="h-7 w-7" />
          </Button>
        </div>
      </AccordionItem>
    {/each}
  </Accordion>

  <!-- Add Note Modal -->
  <Modal
    title="Add New Note"
    bind:open={add_note_modal}
    transition={slide}
    autoclose
    onclose={() => {
      title = "";
      content = "";
    }}
  >
    <Label for="add-title" class="mb-2">Title</Label>
    <Input type="text" id="add-title" bind:value={title} required />

    <Label for="add-content" class="mb-2 mt-4">Content</Label>
    <Textarea
      id="add-content"
      bind:value={content}
      placeholder="Your message"
      rows={4}
    />

    <div class="flex justify-end gap-2 mt-4">
      <Button onclick={addNote}>Save</Button>
      <Button color="alternative" onclick={() => (add_note_modal = false)}
        >Cancel</Button
      >
    </div>
  </Modal>

  <!-- Update Note Modal -->
  <Modal
    title="Update Note"
    bind:open={update_note_modal}
    transition={slide}
    autoclose
    onclose={() => {
      title = "";
      content = "";
      currentNoteId = null;
    }}
  >
    <Label for="update-title" class="mb-2">Title</Label>
    <Input type="text" id="update-title" bind:value={title} required />

    <Label for="update-content" class="mb-2 mt-4">Content</Label>
    <Textarea
      id="update-content"
      bind:value={content}
      placeholder="Your message"
      rows={4}
    />

    <div class="flex justify-end gap-2 mt-4">
      <Button onclick={updateNote}>Save</Button>
      <Button color="alternative" onclick={() => (update_note_modal = false)}
        >Cancel</Button
      >
    </div>
  </Modal>
</main>

<style lang="postcss">
  @reference "tailwindcss";

  :global(html) {
    background-color: theme(--color-slate-100);
  }
</style>
