const { Client, GatewayIntentBits, ApplicationCommandOptionType, ActionRowBuilder, ButtonBuilder, ButtonStyle, EmbedBuilder } = require('discord.js');
const admin = require('firebase-admin');

// ==================== CONFIGURATION BLOCK ====================
const DISCORD_TOKEN = "MTUwNzIyMjQzMDM4Nzc5ODEwNw.GLXNsr.51bb7lYni7N_yv7aYrrnJuKo7MJ3kFlDVHcEAA"; 
const CASHAPP_TAG = "jerbear13ish";
const ADMIN_CHANNEL_ID = "1507231114283188298";
const DATABASE_URL = "https://console.firebase.google.com/u/0/project/d-and-d-campaign-hub/database/d-and-d-campaign-hub-default-rtdb/data";
const PREMIUM_ROLE_NAME = "Premium DM";
// =============================================================

// Initialize Firebase Admin
admin.initializeApp({
    credential: admin.credential.cert(require('./firebase-keys.json')), // Assumes your Firebase key file is named this
    databaseURL: DATABASE_URL
});
const db = admin.database();

// Initialize the Discord Client with necessary intents
const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent
    ]
});

// Define slash commands configuration
const commands = [
    {
        name: 'buy_premium',
        description: 'Get instructions to purchase Premium DM status'
    },
    {
        name: 'submit_receipt',
        description: 'Submit a receipt screenshot for validation',
        options: [
            {
                name: 'transaction_id',
                description: 'Your Cash App Web Receipt ID',
                type: ApplicationCommandOptionType.String,
                required: true
            },
            {
                name: 'screenshot',
                description: 'Upload the screenshot of your receipt',
                type: ApplicationCommandOptionType.Attachment,
                required: true
            }
        ]
    },
    {
        name: 'create_campaign',
        description: 'Create a brand new multiplayer campaign channel',
        options: [
            {
                name: 'title',
                description: 'The title of your campaign',
                type: ApplicationCommandOptionType.String,
                required: true
            }
        ]
    },
    {
        name: 'ping',
        description: 'Check if the bot is online'
    }
];

// Event: Client is ready
client.once('ready', async () => {
    console.log(`Logged in as ${client.user.tag}!`);
    try {
        console.log('Started refreshing application (/) commands globally...');
        await client.application.commands.set(commands);
        console.log('Successfully reloaded application (/) commands globally.');
    } catch (error) {
        console.error('Error registering global commands:', error);
    }
});

// Event: Interaction Create (Command & Button execution)
client.on('interactionCreate', async (interaction) => {
    // Handle Buttons (Admin Approval)
    if (interaction.isButton()) {
        if (!interaction.member.permissions.has('Administrator')) {
            return interaction.reply({ content: 'Only admins can approve payments!', ephemeral: true });
        }

        if (interaction.customId.startsWith('approve_')) {
            const userId = interaction.customId.split('_')[1];
            
            try {
                // Update Firebase database tier status
                await db.ref(`users/${userId}`).update({ isPremium: true });

                // Find user and apply Discord Role
                const member = await interaction.guild.members.fetch(userId);
                const role = interaction.guild.roles.cache.find(r => r.name === PREMIUM_ROLE_NAME);
                if (role) await member.roles.add(role);

                // Send Confirmation DM to user
                await member.send("🎉 **Payment Approved!** Your account has been upgraded to Premium DM status. You now have unlimited campaign slots!").catch(() => null);

                await interaction.update({ content: `✅ **Approved!** ${member.user.tag} has been upgraded to Premium status.`, components: [] });
            } catch (err) {
                console.error(err);
                await interaction.reply({ content: 'Failed to process approval configuration.', ephemeral: true });
            }
        }
        return;
    }

    // Only process slash commands below
    if (!interaction.isChatInputCommand()) return;

    if (!interaction.guild) {
        return interaction.reply({ content: 'This command can only be used within a server.', ephemeral: true });
    }

    const { commandName } = interaction;

    // Handle /buy_premium command
    if (commandName === 'buy_premium') {
        const embed = new EmbedBuilder()
            .setTitle('✨ Upgrade to Premium DM Status')
            .setDescription(`Gain access to unlimited campaigns and master host privileges!\n\n1. Send **$5.00** via Cash App: [Click Here to Pay](https://cash.app/$${CASHAPP_TAG}/5.00)\n2. **CRITICAL:** Put your exact Discord username (\`${interaction.user.tag}\`) in the payment note.\n3. Take a screenshot and type \`/submit_receipt\` down below to finish onboarding.`)
            .setColor('#5865F2');
        return interaction.reply({ embeds: [embed], ephemeral: true });
    }

    // Handle /submit_receipt command
    if (commandName === 'submit_receipt') {
        try {
            const txId = interaction.options.getString('transaction_id');
            const attachment = interaction.options.getAttachment('screenshot');

            if (!attachment) {
                return interaction.reply({ content: 'Error: Please upload a valid image file.', ephemeral: true });
            }

            await interaction.deferReply({ ephemeral: true });

            // Forward Receipt to Moderation Channel
            const adminChannel = client.channels.cache.get(ADMIN_CHANNEL_ID);
            if (adminChannel) {
                const adminEmbed = new EmbedBuilder()
                    .setTitle('💰 Pending Payment Verification')
                    .setDescription(`**User:** ${interaction.user} (${interaction.user.id})\n**Tx ID:** \`${txId}\``)
                    .setImage(attachment.url)
                    .setColor('#FEE75C');

                const row = new ActionRowBuilder().addComponents(
                    new ButtonBuilder()
                        .setCustomId(`approve_${interaction.user.id}`)
                        .setLabel('Confirm & Approve User')
                        .setStyle(ButtonStyle.Success)
                );

                await adminChannel.send({ embeds: [adminEmbed], components: [row] });
            }

            await interaction.editReply({
                content: `Thank you! Your transaction info (\`${txId}\`) has been forwarded to the administration panel for real-world verification.`
            });

        } catch (error) {
            console.error(error);
            await interaction.editReply({ content: 'There was an error processing your configuration logs.' });
        }
    }

    // Handle /create_campaign command
    if (commandName === 'create_campaign') {
        const title = interaction.options.getString('title');
        const userRef = db.ref(`users/${interaction.user.id}`);
        const snapshot = await userRef.once('value');
        const isPremium = snapshot.val()?.isPremium || false;

        // Standard User Lock System Check
        if (!isPremium) {
            const userCampaigns = await db.ref(`campaigns`).orderByChild('ownerId').equalTo(interaction.user.id).once('value');
            if (userCampaigns.exists()) {
                return interaction.reply({ content: '🛑 **Limit Reached!** Standard users can only host 1 active campaign group. Use `/buy_premium` to unlock infinite creation tools!', ephemeral: true });
            }
        }

        // Generate the new server channel asset
        const cleanChannelName = title.toLowerCase().replace(/[^a-z0-9]+/g, '-');
        const channel = await interaction.guild.channels.create({
            name: `🎲-${cleanChannelName}`,
            type: 0 // Text Channel
        });

        // Log Campaign Reference ID mapping straight to Firebase
        await db.ref(`campaigns/${channel.id}`).set({
            title: title,
            ownerId: interaction.user.id,
            createdAt: Date.now()
        });

        await interaction.reply({ content: `⚔️ **Adventure Ready!** Your campaign room has been successfully generated here: ${channel}` });
    }

    // Handle /ping command
    if (commandName === 'ping') {
        await interaction.reply({ content: 'Pong!', ephemeral: true });
    }
});

client.login(MTUwNzIyMjQzMDM4Nzc5ODEwNw.GLXNsr.51bb7lYni7N_yv7aYrrnJuKo7MJ3kFlDVHcEAA);