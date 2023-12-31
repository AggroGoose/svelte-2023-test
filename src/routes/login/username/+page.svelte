<script lang="ts">
  import AuthCheck from "$lib/components/AuthCheck.svelte";
  import { db, user, userData } from "$lib/firebase";
  import { doc, getDoc, writeBatch } from 'firebase/firestore'; 

  let username = "";
  let loading = false;
  let isAvailable = false;
  let debounceTimer: NodeJS.Timeout;

  const re = /^(?=[a-zA-Z0-9._]{3, 16}$)(?!.*[_.]{2})[^_.].*[^_.]$/;

  $: isValid = username?.length > 2 && username.length < 16 && re.test(username);
  $: isTouched = username.length > 0;
  $: isTaken = isValid && !isAvailable && !loading;

  async function checkAvailability() {
    isAvailable = false;
    clearTimeout(debounceTimer);

    loading = true;

    debounceTimer = setTimeout(async () => {
        console.log("checking availability of", username);

        const ref = doc(db, "usernames", username);
        const exists = await getDoc(ref).then((doc) => doc.exists());

        isAvailable = !exists;
        loading = false;
    }, 500);
    
  }

  async function confirmUsername() {
    console.log("confirming username", username);
    const batch = writeBatch(db);
    batch.set(doc(db, 'usernames', username), {uid: $user?.uid});
    batch.set(doc(db, 'users', $user!.uid), {
        username,
        photoURL: $user?.photoURL ?? null,
        published: true,
        bio: 'I am the Walrus',
        links: [
            {
                title: 'Test Link',
                url: 'https://kung.foo',
                icon: 'custom'
            }
        ]
    });

    await batch.commit()

    username='';
    isAvailable = false;
  }
</script>

<AuthCheck>
    {#if $userData?.username}
        <p>
            Your username is <span>@{$userData.username}</span>
        </p>
        <p>(Usernames cannot be changed)</p>
        <a href="/login/photo">Upload Profile Image</a>
    {:else}
        <h2>Username</h2>
        <form class="w-2/5" on:submit|preventDefault={confirmUsername}>
            <input
                type="text"
                placeholder="Username"
                class="input w-full"
                class:input-error={(!isValid && isTouched)}
                class:input-warning={isTaken}
                class:input-success={isAvailable && isValid && !loading}
                bind:value={username}
                on:input={checkAvailability}
            />

            <p>Is available? {isAvailable}</p>

            <button class="btn btn-success">Confirm username @{username}</button>
        </form>
    {/if}
</AuthCheck>