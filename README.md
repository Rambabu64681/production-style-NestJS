# production style NestJS



import {
  BadRequestException,
  Body,
  ConflictException,
  Controller,
  HttpCode,
  HttpStatus,
  Injectable,
  InternalServerErrorException,
  Module,
  Post,
  Headers,
  ValidationPipe,
} from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import {
  IsNotEmpty,
  IsNumber,
  IsPositive,
  IsString,
  Length,
} from 'class-validator';
import { randomUUID } from 'crypto';

/* =========================
   DTOs
========================= */

class CreateTransactionDto {
  @IsString()
  @IsNotEmpty()
  @Length(10, 20)
  senderAccount: string;

  @IsString()
  @IsNotEmpty()
  @Length(10, 20)
  receiverAccount: string;

  @IsNumber()
  @IsPositive()
  amount: number;

  @IsString()
  @IsNotEmpty()
  @Length(3, 3)
  currency: string;
}

class TransactionResponseDto {
  transactionId: string;
  status: 'SUCCESS' | 'FAILED';
  message: string;
  createdAt: Date;
}

/* =========================
   Internal Interface
========================= */

interface TransactionRecord {
  transactionId: string;
  senderAccount: string;
  receiverAccount: string;
  amount: number;
  currency: string;
  status: 'SUCCESS' | 'FAILED';
  createdAt: Date;
  idempotencyKey: string;
}

/* =========================
   Service
========================= */

@Injectable()
class TransactionsService {
  private readonly transactions: Map<string, TransactionRecord> = new Map();
  private readonly idempotencyStore: Map<string, TransactionResponseDto> =
    new Map();

  async createTransaction(
    dto: CreateTransactionDto,
    idempotencyKey: string,
  ): Promise<TransactionResponseDto> {
    const existingResponse = this.idempotencyStore.get(idempotencyKey);
    if (existingResponse) {
      return existingResponse;
    }

    if (dto.senderAccount === dto.receiverAccount) {
      throw new ConflictException(
        'Sender and receiver accounts cannot be the same',
      );
    }

    try {
      const transactionId = randomUUID();

      const transaction: TransactionRecord = {
        transactionId,
        senderAccount: dto.senderAccount,
        receiverAccount: dto.receiverAccount,
        amount: dto.amount,
        currency: dto.currency,
        status: 'SUCCESS',
        createdAt: new Date(),
        idempotencyKey,
      };

      this.transactions.set(transactionId, transaction);

      const response: TransactionResponseDto = {
        transactionId,
        status: transaction.status,
        message: 'Transaction processed successfully',
        createdAt: transaction.createdAt,
      };

      this.idempotencyStore.set(idempotencyKey, response);

      return response;
    } catch (error) {
      throw new InternalServerErrorException('Failed to process transaction');
    }
  }
}

/* =========================
   Controller
========================= */

@Controller('transactions')
class TransactionsController {
  constructor(private readonly transactionsService: TransactionsService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async createTransaction(
    @Body() createTransactionDto: CreateTransactionDto,
    @Headers('idempotency-key') idempotencyKey: string,
  ): Promise<TransactionResponseDto> {
    if (!idempotencyKey) {
      throw new BadRequestException('Idempotency-Key header is required');
    }

    return this.transactionsService.createTransaction(
      createTransactionDto,
      idempotencyKey,
    );
  }
}

/* =========================
   Module
========================= */

@Module({
  controllers: [TransactionsController],
  providers: [TransactionsService],
})
class AppModule {}

/* =========================
   Bootstrap
========================= */

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(3000);
  console.log('Application running at http://localhost:3000');
}

bootstrap();
