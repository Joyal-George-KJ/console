<script lang="ts">
    import { invalidate } from '$app/navigation';
    import { Modal, Card } from '$lib/components';
    import { Repositories } from '$lib/components/git';
    import { Dependencies } from '$lib/constants';
    import { Link } from '$lib/elements';
    import { Button, InputCheckbox } from '$lib/elements/forms';
    import { timeFromNow } from '$lib/helpers/date';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { installation, repository, sortBranches } from '$lib/stores/vcs';
    import { Runtime, VCSDeploymentType, type Models } from '@appwrite.io/console';
    import { IconGithub } from '@appwrite.io/pink-icons-svelte';
    import { Icon, Input, Layout, Skeleton, Typography } from '@appwrite.io/pink-svelte';
    import { func } from '../store';
    import { page } from '$app/state';

    export let show = false;

    export let installations: Models.InstallationList;
    let hasRepository = !!$func?.providerRepositoryId;
    let selectedRepository: string = $func.providerRepositoryId;
    let branch: string = null;
    let commit: string = null;
    let activate = true;
    let error = '';

    async function loadInstallations() {
        if (!$func?.installationId && installations?.total > 0) {
            installation.set(installations.installations[0]);
        }
        $installation = installations.installations.find(
            (installation) => installation.$id === $func.installationId
        );
        if (!$installation?.$id) {
            $installation = installations.installations[0];
        }
    }

    async function load() {
        try {
            await loadInstallations();
            if (!$repository?.id && hasRepository) {
                $repository = await sdk
                    .forProject(page.params.region, page.params.project)
                    .vcs.getRepository($installation.$id, $func.providerRepositoryId);
            }
            selectedRepository = $repository?.id;
            const branchList = await sdk
                .forProject(page.params.region, page.params.project)
                .vcs.listRepositoryBranches($installation.$id, selectedRepository);

            const sorted = sortBranches(branchList.branches);

            branch = sorted.find((b) => b.name === $func.providerBranch)
                ? $func.providerBranch
                : (sorted[0]?.name ?? null);

            if (!branch) {
                branch = 'main';
            }

            return sorted;
        } catch {
            return;
        }
    }

    async function createDeployment() {
        try {
            if (!$func?.providerRepositoryId) {
                await sdk
                    .forProject(page.params.region, page.params.project)
                    .functions.update(
                        $func.$id,
                        $func.name,
                        $func.runtime as Runtime,
                        $func.execute || undefined,
                        $func.events || undefined,
                        $func.schedule || undefined,
                        $func.timeout || undefined,
                        $func.enabled || undefined,
                        $func.logging || undefined,
                        $func.entrypoint,
                        $func.commands || undefined,
                        $func.scopes || undefined,
                        $installation.$id || undefined,
                        selectedRepository || undefined,
                        branch || undefined,
                        undefined,
                        undefined,
                        undefined
                    );
            }
            if (commit) {
                await sdk
                    .forProject(page.params.region, page.params.project)
                    .functions.createVcsDeployment(
                        $func.$id,
                        VCSDeploymentType.Commit,
                        commit,
                        activate
                    );
            } else if (branch) {
                await sdk
                    .forProject(page.params.region, page.params.project)
                    .functions.createVcsDeployment(
                        $func.$id,
                        VCSDeploymentType.Branch,
                        branch,
                        activate
                    );
            }
            show = false;
            invalidate(Dependencies.DEPLOYMENTS);
            addNotification({
                message: activate
                    ? 'Deployment is in progress. It will be automatically activated after build step completes.'
                    : 'Deployment is in progress. You can activate it after build step completes.',
                type: 'success'
            });
        } catch (e) {
            error = e.message;
        }
    }

    $: if (!show) {
        error = '';
        branch = null;
        commit = null;
        activate = true;
        hasRepository = !!$func?.providerRepositoryId;
    }
</script>

<Modal title="Create Git deployment" bind:show onSubmit={createDeployment} bind:error>
    <span slot="description">
        Enter a valid commit reference to create a new deployment. <Link
            href="https://appwrite.io/docs/products/functions/deployments#create-deployment"
            external>Learn more</Link>
    </span>
    {#if hasRepository}
        {#await load()}
            <Card padding="xs" radius="s" variant="secondary">
                <Layout.Stack
                    direction="row"
                    justifyContent="space-between"
                    alignItems="center"
                    gap="xs">
                    <Layout.Stack direction="row" alignItems="center" gap="s">
                        <Icon size="s" icon={IconGithub} />
                        <Skeleton variant="line" width={100} height={19.6} />
                    </Layout.Stack>
                </Layout.Stack>
            </Card>
            <Layout.Stack gap="s">
                <Skeleton variant="line" width={100} height={19.6} />
                <Skeleton variant="line" width={350} height={31} />
            </Layout.Stack>
        {:then branches}
            {@const options =
                branches
                    ?.map((branch) => {
                        return {
                            value: branch.name,
                            label: branch.name
                        };
                    })
                    ?.sort((a, b) => {
                        return a.label > b.label ? 1 : -1;
                    }) ?? []}
            <Card padding="xs" radius="s" variant="secondary">
                <Layout.Stack
                    direction="row"
                    justifyContent="space-between"
                    alignItems="center"
                    gap="xs">
                    <Layout.Stack direction="row" alignItems="center" gap="s" inline>
                        <Icon icon={IconGithub} />
                        <Link
                            external
                            href={`https://github.com/${$repository?.organization}/${$repository?.name}`}>
                            <Layout.Stack direction="row" alignItems="center" gap="s" inline>
                                {$repository?.organization}/{$repository?.name}
                            </Layout.Stack>
                        </Link>
                    </Layout.Stack>
                    <Typography.Caption variant="400" color="--fgcolor-neutral-tertiary">
                        Last updated {timeFromNow($repository?.pushedAt)}
                    </Typography.Caption>
                </Layout.Stack>
            </Card>
            <Input.ComboBox
                required
                id="branch"
                label="Production branch"
                placeholder="Select branch"
                bind:value={branch}
                on:select={(event) => {
                    branch = event.detail.value;
                }}
                {options} />
            <!-- <InputText
                id="commit"
                label="Commit hash"
                placeholder="Select commit"
                bind:value={commit} /> -->
            {#if branch}
                <InputCheckbox
                    label="Activate deployment after build"
                    id="activate"
                    description="This deployment will automatically activate after the build completes. If
                unchecked, it will remain inactive, and you can activate it manually later."
                    bind:checked={activate} />
            {/if}
        {/await}
    {:else}
        <Repositories
            bind:selectedRepository
            installationList={installations}
            product="functions"
            action="button"
            callbackState={{
                createDeployment: 'true'
            }}
            connect={(e) => {
                repository.set(e);
                hasRepository = true;
            }} />
    {/if}
    <svelte:fragment slot="footer">
        <Button text size="s" on:click={() => (show = false)}>Cancel</Button>
        <Button submit size="s" disabled={!$installation?.$id || !selectedRepository || !branch}>
            Create
        </Button>
    </svelte:fragment>
</Modal>
