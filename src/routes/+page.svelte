<script lang="ts">
	import { supabase } from '$lib/supabaseClient';
    import { onMount } from 'svelte';

	let newTask = '';
	let email = '';
	let description = '';
    let dueDate = '';
	let status = 'Pending';
	let saving = false;
	let message = '';
	let searchQuery = '';


	let tasks: any[] = [];
    let filteredTasks: any[] = [];
    let priority = 'Medium';
	let filter: 'all' | 'completed' | 'pending' | 'high' = 'all';

	let user: any = null;
    let editingId: string | null = null;
	let editedTitle = '';
	let editedPriority = 'Medium';
	let editedDueDate = '';
	


    // Check user session
	onMount(async () => {
		const { data } = await supabase.auth.getUser();
		user = data.user;
		if (user) {
			await loadTasks();
		}
	});

	// Check login
	supabase.auth.getUser().then(({ data }) => {
		user = data.user;
		if (user) loadTasks();
	});

	// Login
	async function login() {
		const { error } = await supabase.auth.signInWithOtp({ email });
		if (error) alert(error.message);
		else alert('Check your email for login link');
	}

	// Logout
	async function logout() {
		await supabase.auth.signOut();
		user = null;
		tasks = [];
	}

	// Load tasks
	async function loadTasks() {
		if(!user) return;

		const { data, error } = await supabase
			.from('tasks')
			.select('*')
			.eq('user_id', user.id)
			.order('created_at', { ascending: false });

		if (!error) tasks = data ?? [];
	}

    // Apply filters
	$: filteredTasks = tasks
	.filter(t => {
		if (filter === 'completed') return t.completed;
		if (filter === 'pending') return !t.completed;
		if (filter === 'high') return t.priority === 'High';
		return true;
	})
	.filter(t =>
		t.title.toLowerCase().includes(searchQuery.toLowerCase())
	);


	// Add task
	async function addTask() {
		if (!newTask.trim()) {
			message = 'Title is required';
			return;
		}

		saving = true;
		message = '';

		await supabase.from('tasks').insert({
			title: newTask,
			description,
			due_date: dueDate || null,
			status,
			priority,
			user_id: user.id
		});


		newTask = '';
		description = '';
		dueDate = '';
		status = 'Pending';
		loadTasks();

		saving = false;
	}

	// Delete task
	async function deleteTask(id: string) {
		await supabase.from('tasks').delete().eq('id', id);
		loadTasks();
	}

	// Toggle completed
	async function toggleTask(task: any) {
		const newCompleted = !task.completed;

		await supabase
			.from('tasks')
			.update({
				completed: newCompleted,
				status: newCompleted ? 'Completed' : 'Pending'
			})
			.eq('id', task.id);

		loadTasks();
	}

    // Start editing
	function startEdit(task: any) {
		editingId = task.id;
		editedTitle = task.title;
		editedPriority = task.priority;
		editedDueDate = task.due_date || '';
	}

	// Cancel editing
	function cancelEdit() {
		editingId = null;
		editedTitle = '';
	}

	// Save edited title
	async function saveEdit(id: string) {
		if (!editedTitle.trim()) return;

		await supabase
			.from('tasks')
			.update({ 
				title: editedTitle,
				priority: editedPriority,
				due_date: editedDueDate || null 
			})
			.eq('id', id);

		cancelEdit();
		loadTasks();
	}

	
</script>

<h1><b>Task Manager</b></h1><br>

{#if !user}
	<div class="min-h-screen flex items-center justify-center bg-gray-100">
		<div class="bg-white p-6 rounded shadow text-center">
			<h2 class="text-xl font-bold mb-4">Please login to continue</h2>
            <input
                type="email"
                placeholder="Enter email"
                bind:value={email}
            />
            <button on:click={login}>Login</button>
			<p class="text-gray-500">Check your email for the magic link</p>
		</div>
	</div>
{:else}
	<div class="min-h-screen bg-gray-100 p-6">
		<div class="max-w-xl mx-auto bg-white rounded-xl shadow-md p-6">

			<h1 class="text-2xl font-bold mb-4 text-center">Task Manager</h1>

			<button
				class="mb-4 text-sm text-red-500 hover:underline"
				on:click={logout}>
				Logout
			</button>

			<div class="flex gap-2 mb-4">
				<input
					class="flex-1 border rounded px-3 py-2 h-[42px]"
					placeholder="Task title"
					bind:value={newTask} />

				<textarea
					class="flex-1 border rounded px-3 py-2"
					placeholder="Task description"
					bind:value={description}>
				</textarea>

				<select
					class="border px-3 py-2 rounded"
					bind:value={status}>
					<option>Pending</option>
					<option>In Progress</option>
					<option>Completed</option>
				</select>

				<input
					type="date"
					class="border rounded px-3 py-2"
					bind:value={dueDate} />

                <select
                    class="border px-3 py-2 rounded"
                    bind:value={priority}>
                    <option>High</option>
                    <option>Medium</option>
                    <option>Low</option>
			    </select>

				<button
					class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
					on:click={addTask}>
					Add
				</button>
			</div>

            <!-- Filters -->
            <div class="flex gap-2 mb-4">
			<button on:click={() => filter = 'all'}>All</button>
			<button on:click={() => filter = 'completed'}>Completed</button>
			<button on:click={() => filter = 'pending'}>Pending</button>
			<button on:click={() => filter = 'high'}>High Priority</button>
		</div>
		<!-- Search -->
		<input
			class="w-full border rounded px-3 py-2 mb-4"
			type="text"
			placeholder="Search tasks..."
			bind:value={searchQuery} />

			<ul class="space-y-3">
				{#each filteredTasks as task}
					<li class="flex justify-between items-center bg-gray-100 p-2 rounded">

						<div class="flex items-center gap-2">
							<input
								type="checkbox"
								bind:checked={task.completed}
								
							/>

							{#if editingId === task.id}
								<div class="flex gap-2 items-center">
									<input
										class="border rounded px-2 py-1"
										bind:value={editedTitle} />

									<select
										class="border rounded px-2 py-1"
										bind:value={editedPriority}>
										<option>High</option>
										<option>Medium</option>
										<option>Low</option>
									</select>

									<input
										type="date"
										class="border rounded px-2 py-1"
										bind:value={editedDueDate} />

									<button class="text-green-600" on:click={() => saveEdit(task.id)}>
										Save
									</button>

									<button class="text-gray-500" on:click={cancelEdit}>
										Cancel
									</button>
								</div>

							{:else}
								<span class={task.completed ? 'line-through text-gray-400' : ''}>
									{task.title}
								</span>
								<button
									class="text-blue-500 hover:underline"
									on:click={() => startEdit(task)}>
									✏️
								</button>
							{/if}
                            
                            <span class="text-xs px-2 py-1 rounded bg-gray-200">
							    {task.priority}
						    </span>

							{#if task.due_date}
								<span class="text-xs text-gray-500">
									📅 {task.due_date}
								</span>
							{/if}
						</div>

						<button
							class="text-red-500 hover:underline"
							on:click={() => deleteTask(task.id)}>
							❌
						</button>
					</li>
				{/each}
			</ul>

		</div>
	</div>
{/if}